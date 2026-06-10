# One Turn — Start to Finish

What happens when you send a message, tap Continue, or skip time.

---

## Overview diagram

```text
YOU (Flutter app)
  │
  │  WebSocket: chat / continue / replay
  ▼
API server (thin)
  │  Validates session, NSFW consent
  │  Enqueues job — does NOT call the AI here
  ▼
Generation worker
  │  1. Build context packet (search + codex + time + place)
  │  2. Build prompt (bounded sections)
  │  3. Stream AI reply → you see text live
  │  4. Extract scene metadata (place, time passed, scene type)
  │  5. Save event to Mongo
  │  6. Update codex + entity graph
  │  7. Enqueue memory job + maybe summary job
  ▼
Memory worker (async, ~1s later)
  │  Extract 0–3 memory atoms, embed, save
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

**Server entry:** `play-ws.service.ts` → `generation.service.ts` → `dispatch()`

Dispatch is intentionally **thin**: load session, check consent, enqueue a `generate` job. No RAG here anymore.

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

### 4. AI streams the reply

**File:** `generation.processor.ts`

- Streams tokens to app as `generation_delta`
- Finishes with `generation_complete` / `generation_stream_end`
- Runs prose hygiene repair if needed

You see the bubble fill in live on `NarrativeBubble`.

### 5. Server extracts what happened (metadata)

**File:** `worker/lib/metadata-extractor.ts`

After prose is done, a smaller LLM call extracts:

- `scene_tag` (dialogue, combat, romantic, intimate, exploration, …)
- `present_characters` (who was in the scene)
- End-of-turn **location** (if the story moved)
- **Time elapsed** ("three days later", "weeks passed")
- Location state changes and permanent place facts
- Suggested **choice chips** for your next turn (optional)

### 6. Event saved and world updated

**File:** `generation.processor.ts`

A new **event** document is written with:

- Your input + AI response
- State/flag mutations (if gamified stats exist)
- **Codex deltas** (ledgered on the event for rewind)
- `time_anchor`, `location_anchor`
- Maybe `type: 'travel'` if location changed concretely
- Maybe `data.time_advanced` if narration skipped time

Instance cursors updated:
- `current_time_anchor` (calendar day advances)
- `current_location`
- `current_scene` (tag + turn count in scene)
- Bond meters via codex

### 7. Async: memories created

**Queue:** `memory-curation` → `memory.processor.ts`

~1 second later:

1. LLM extracts 0–3 **memory atoms** (rich: emotions, subjects, threads)
2. Resolves names to **entity IDs**
3. Embeds text → Pinecone `mem_{instanceId}`
4. Saves to Mongo with provenance (`source_event_ids`)
5. Relationship-type memories create **graph edges**
6. WebSocket pushes `memories_curated` — app adds to local list (max 50)

### 8. Async: maybe a scene summary

If this turn completes a **12-turn block** in the same scene tag:

- `scene-summary` queue runs
- 12 turns compressed to 2 paragraphs
- Every 8 scenes → **chapter** summary
- Every 4 chapters → **arc** summary
- Summaries embedded in Pinecone `sum_{instanceId}` for later retrieval

---

## Continue (no message)

**WebSocket:** `continue`

- Same pipeline but user message is empty / "continue"
- Skips RAG query text (uses continuation framing)
- Scene may advance autonomously

## Time skip

**App:** `AdvanceTimeButton` → sheet (hours / day / season)

**WebSocket:** `continue` with `{ advance: "day" }` etc.

- Event type often `calendar_tick`
- Calendar advances deterministically
- May weave in an open thread as "fate" on long skips

## Replay a turn

**WebSocket:** `replay` with `{ event_id }`

- Generates alternative AI responses (variants)
- Selecting a variant re-curates memories for that event
- Choice chips clear so stale suggestions don't linger

## Rewind

**REST:** `POST /chronicle/rewind/:instanceId`

- Deletes events at/after chosen sequence
- Deletes sourced memories + summary vectors
- Rebuilds codex from surviving event deltas
- Repairs entity graph + location facts
- Replays world state from template defaults
- App reloads via `load_instance`

---

## Side chat (private character conversation)

Separate path — see [04 — Frontend map](04-frontend-where-things-live.md).

- Does **not** advance main story time or location
- Does **not** appear in Play timeline
- Memories scoped to who was in the chat
- Codex still updates for that character (relationship meters)

---

## What you feel as a player

| Moment | What you see |
|--------|--------------|
| Send message | Streaming gold prose |
| Turn done | Maybe new choice chips; bond rings may shift |
| ~1s later | New memories available (tap names in text) |
| Every 12 turns | Invisible — older turns summarized for future recall |
| Ask about old character | Memories + pinned codex card surface together |

The system is designed so **play feels continuous** while **memory work happens mostly off the hot path**.
