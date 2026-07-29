# One Turn — Start to Finish

What happens when you send a message, tap Continue, skip time, travel via World Actions, or replay.

---

## Overview diagram

```text
YOU (Flutter app)
  │
  │  WebSocket: chat / continue / world_action / replay
  ▼
API server (thin)
  │  Validates session, NSFW consent
  │  billingService.reserve(Ink) for story turns
  │  Enqueues job — does NOT call the AI here
  ▼
Generation worker
  │  1. Build context packet (search + codex + time + place)
  │  2. Build prompt (bounded sections)
  │  3. Stream AI reply as PROSE (generation_delta) — not a structured JSON turn body
  │  4. Post-prose pipeline (below)
  │  5. Save event to Mongo; update cursors / party
  │  6. Fire-and-forget: graph, kinship, candidates, signal_ledger, …
  │  7. Enqueue memory job + maybe summary job
  │  8. billingService.settle (or release on final pre-stream fail)
  ▼
Memory worker (async, ~1s later)
  │  Extract 0–3 memory atoms, embed, save
  │  Materialize updates_memory_ids when supersession marked
  │  Push memories_curated to your app
  ▼
Summary worker (every 12 turns in same scene)
  │  Compress block into 2 paragraphs
  │  Maybe roll up into chapter → arc
```

---

## Step by step: you type and send

### 1. App sends your message

**File:** `everlore/lib/features/play/state/play_cubit.dart`

- WebSocket action: `chat` with `{ message }`
- Your text can mix spoken dialogue and `*actions in asterisks*` — the server parses these separately

**Server entry:** `play-ws.service.ts` → **reserve Ink** → `generation.service.ts` → `dispatch()`

Dispatch is intentionally **thin**: load session, check consent, enqueue a `generate` job. No RAG here.

### 2. Worker builds the briefing (context packet)

**File:** `context-packet.service.ts` → `buildContextPacket()`

Order matters:

1. Load last **6** main-story events (side chats excluded)
2. Load latest **scene summary** for turns before that window
3. Read **current story time** and **current location** from the instance
4. Find characters **you named** in your message → pin their cards
5. Run **hybrid search** (vector + keywords + entity neighborhood + place + timeline filter)
6. In parallel, search **distant chapter/scene summaries** semantically
7. **After** search: rank codex pool (40 → 16), pin cards for entities found in memories
8. Package everything into a `ContextPacket`

### 3. Worker builds the prompt

**File:** `prompt-builder.ts` → `buildPrompt()`

Layers injected into the AI:

| Section | Source |
|---------|--------|
| World identity + voice + rules | Static (cacheable) |
| Chat mode + reply length | Dynamic |
| Protagonist / player block | Codex |
| NPC character cards (ranked + pinned) | Codex |
| Kinship brief / relatives (when relevant) | Kinship graph |
| Current place + story date + timeline | Context packet |
| World stats & flags | Instance (hidden if stat-less) |
| Retrieved lore | Pinecone template namespace |
| Retrieved memories | Hybrid RAG |
| Open threads (unresolved promises) | Mongo |
| Relevant past chapters | Summary search |
| Scene summary | Mongo |
| Recent turns | Last 6, token-budgeted |
| POV reminder + your current turn | Last messages |

Hard rule: recent turns get a **1000-token floor** — reference sections can't starve them.

### 4. AI streams the reply (prose, not JSON)

**File:** `generation.processor.ts` + `prose-stream-filter.ts` / `choice-tail.ts`

- Streams tokens to app as `generation_delta` — **player-facing narration is prose**
- Accidental JSON envelopes are stripped from the stream
- Optional choice chips are parsed from a `==CHOICES==` tail off the stream (not a structured turn response schema)
- Finishes with `generation_complete` / `generation_stream_end`
- Runs prose hygiene repair if needed

You see the bubble fill in live on `NarrativeBubble`.

### 5. Post-prose: what the server extracts and commits

After prose is done (not as part of the streamed body):

| Step | Module / service | What |
|------|------------------|------|
| Scene metadata | `metadata-extractor.ts` | `scene_tag`, `present_characters`, end location, time elapsed, place facts, suggested choices |
| Deterministic signals | `movement-signal`, `time-skip-signal`, `party-signal`, kinship pattern extractors, presence-gap | Backstops when the witness under-reports |
| **Entity adjudication** | `entity-adjudicator.ts` | Semantic judge for strong person-candidate terms only |
| Persist event | `generation.processor.ts` | Event + travel / time / codex_deltas ledgers |
| Fold cursors | processor | Location, calendar, scene, party, bond meters |
| Kinship apply | `kinship-graph.service` + worker extractors | Structural family edges from assertions / transitions |
| Relation candidates | `relation-candidate-detector` / `canon-revision-detector` | Narrator proposals queued for player review (not canon) |
| **Signal ledger** | `signal-ledger.ts` → `signal_ledger` collection | FP/FN tallies for movement / time / party / kinship / presence |
| Anomalies | `projection-anomaly-detector` | Fire-and-forget inconsistency log |
| **Billing settle** | `billing.service.settle` | Consumes the reservation after success / visible-stream failure paths |

### 6. Event saved and world updated

A new **event** document is written with:

- Your input + AI response (prose)
- State/flag mutations (if gamified stats exist)
- **Codex deltas** (ledgered on the event for rewind)
- `time_anchor`, `location_anchor`
- Maybe `type: 'travel'` if location changed concretely
- Maybe `data.time_advanced` if narration skipped time

Instance cursors updated: `current_time_anchor`, `current_location`, `current_scene`, party / bond meters.

### 7. Async: memories created

**Queue:** `memory-curation` → `memory.processor.ts`

~1 second later:

1. LLM extracts 0–3 **memory atoms**
2. Resolves names to **entity IDs**
3. Embeds text → Pinecone `mem_{instanceId}`
4. Saves to Mongo with provenance; may materialize **`updates_memory_ids`** from supersession marks (Phase 2 Slice 1)
5. Relationship-type memories create **graph edges**
6. WebSocket pushes `memories_curated` — app adds to local list (max 50)

### 8. Async: maybe a scene summary

If this turn completes a **12-turn block** in the same scene tag → scene / chapter / arc rollups as before.

---

## Continue (no message)

**WebSocket:** `continue` (from Continue button or World Actions)

- Same pipeline but user message is empty / "continue"
- Skips RAG query text (uses continuation framing)

## Time skip

**App:** World Actions → time pills (or Continue with advance)

**WebSocket:** `continue` with `{ advance: "day" }` etc.

- Event type often `calendar_tick`
- Calendar advances deterministically

## Travel (World Actions)

**WebSocket:** `world_action` with `{ kind: "travel", destination, companions, time_advance? }`

- Structured, server-validated before narration — not inferred from free prose alone

## Kinship (World Actions)

**REST:** `POST /chronicle/kinship/:instanceId` (confirm / correct)

- Authorial write; may also appear as WS `world_action` `{ kind: "relationship", … }` on the server protocol
- Relation **candidates** are reviewed separately via `/chronicle/relation-candidates` + resolve

## Replay a turn

**WebSocket:** `replay` with `{ event_id }`

- Generates alternative AI responses (variants)
- Selecting a variant re-curates memories for that event

## Rewind

**REST:** `POST /chronicle/rewind/:instanceId`

- Deletes events at/after chosen sequence
- Deletes sourced memories + summary vectors
- Rebuilds codex / kinship from surviving ledgers
- Repairs entity graph + location facts
- App reloads via `load_instance`

---

## Side chat (private character conversation)

Separate path — see [04 — Frontend map](04-frontend-where-things-live.md).

- Does **not** advance main story time or location
- Does **not** appear in Play timeline
- Memories scoped to who was in the chat
- Codex still updates for that character (relationship meters)

---

## Billing on the turn path

| Moment | What |
|--------|------|
| Before enqueue | `reserve` — fails with insufficient Ink / rate limits as WS error |
| Job completed / visible stream then fail | `settle` — player saw prose; do not refund that scene |
| Final fail before visible stream | `release` — Ink returned |

Details: [BILLING.md](../server/BILLING.md).

---

## What you feel as a player

| Moment | What you see |
|--------|--------------|
| Send message | Streaming gold prose |
| Turn done | Maybe new choice chips; bond rings may shift; World Actions may show new story details |
| ~1s later | New memories available (tap names in text) |
| Every 12 turns | Invisible — older turns summarized for future recall |
| Ask about old character | Memories + pinned codex card surface together |

The system is designed so **play feels continuous** while **memory work, kinship, candidates, and signal metrics happen mostly off the hot path**.
