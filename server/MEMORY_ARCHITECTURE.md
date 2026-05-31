# Everlore Server — Memory & Codex Architecture

How Everlore keeps a story coherent over thousands of turns without an
ever-growing prompt. This document covers the four context layers (codex,
memories, scene summaries, recent turns), how each is built and retrieved, how
they relate (and notably how they **don't**), and the mechanisms that keep them
accurate over a long playthrough.

---

## 1. The core problem

After 10,000 turns, the full transcript is millions of tokens — far beyond any
model's context window and far too expensive to send each turn. Yet the AI must
still feel like it *remembers* everyone and everything.

**The solution: never send the whole history. Each turn, assemble a small,
fixed-size briefing from four bounded layers.** Whether you are on turn 50 or
turn 50,000, the prompt stays roughly the same size — so **cost per turn is
O(1), not O(turns)**. That bounded-prompt property is the heart of the design.

---

## 2. The four context layers

Every turn's prompt (`src/utils/prompt-builder.ts`) is assembled from:

| Layer | Analogy | Store | Retrieval | Bounded by |
|-------|---------|-------|-----------|------------|
| **Codex** | Cast list / character sheets | `characters` (Mongo) | Importance ranking, top 16 | top-K cap |
| **Memories** | Searchable fact-file | `memories` (Mongo) + Pinecone `mem_{instanceId}` | Semantic (vector) top-K | top-K cap |
| **Scene summaries** | Chapter recaps | `scene_summaries` (Mongo) | Positional (latest before window) | 1 summary |
| **Recent turns** | Last few messages | `events` (Mongo) | Sliding window | token budget |

Plus the static **world lore** (`global_lore` on the template, embedded in
Pinecone `lore_{templateId}`) which is shared across all instances and never
changes per playthrough.

---

## 3. Data stores and what links them

```
                       ┌──────────────────────────────────────┐
   One turn  ─────────▶│  EVENT (events collection)           │
                       │   data.player_input + data.ai_response│
                       └───────────┬───────────────┬──────────┘
                                   │               │
            memory-curation queue  │               │  inline codex extraction
            (worker/processors/    │               │  (worker/processors/
             memory.processor.ts)  │               │   generation.processor.ts)
                                   ▼               ▼
                    ┌──────────────────────┐  ┌──────────────────────┐
                    │  MEMORIES            │  │  CHARACTERS (codex)   │
                    │  Mongo + Pinecone    │  │  Mongo only           │
                    │  source_event_ids →  │  │  first/last_seen_     │
                    │     event            │  │    sequence → event   │
                    │  pinecone_id → vector│  │  (no event-id array)  │
                    └──────────────────────┘  └──────────────────────┘
```

### The important subtlety: memories and characters are NOT directly linked

There is **no foreign key** between a memory and a character. Search the schema
and you will not find `character_id` on a memory or `memory_ids` on a character.

- A **memory** (`src/models/memory.model.ts`) links to its source via
  `source_event_ids: ObjectId[]` and to its vector via `pinecone_id`.
- A **character** (`src/models/character-profile.model.ts`) links to events only
  loosely via `first_seen_sequence` / `last_seen_sequence`.

They are **parallel projections of the same event** — two filing systems fed by
the same source turn. The only connections between them are:

1. **Shared source event** — both were extracted from the same turn (matching
   sequence numbers), but this is implicit, not a stored join.
2. **Shared name / text** — a memory's free text says "Elara…" and there is a
   codex card named "Elara". This *textual* coincidence is the real connective
   tissue, not a pointer.
3. **A computed semantic join (on demand)** — supersession (§7) embeds a
   character fact and finds similar memory vectors. This link is *calculated in
   vector space at that moment*, used, and discarded — never stored.

> **In one line:** events are the structural link, names are the textual link,
> embeddings are the computed link. There is no stored character↔memory relation.

---

## 4. From one event to two stores

When a turn is generated (`generation.processor.ts`), after the narrative is
streamed and the event is inserted, two independent pipelines fire:

1. **Memory curation** is enqueued on the `memory-curation` queue (delayed, low
   priority). `memory.processor.ts` extracts durable facts from the turn,
   embeds each, and writes to **Mongo + Pinecone**.
2. **Codex extraction** runs inline in a fire-and-forget block.
   `extractCharacterCodexDeltas` (gpt-4o-mini) reads the turn + existing cards
   and `characterCodexService.applyDeltas` merges the result into **Mongo
   `characters`**.

Both read the same `player_input + ai_response`; they never read each other.

---

## 5. The Codex (characters) in depth

The codex is the **cast list** — canonical, structured, always-on. It is **not
vectorized**; cards are injected directly into the prompt as hard constraints
("CANONICAL CHARACTER CODEX — never contradict these facts") and reloaded fresh
every turn.

### 5.1 Card shape (`CharacterProfileDoc`)

`canonical_name`, `name_normalized`, `aliases[]`, `role`, `appearance`,
`persona`, `immutable_facts[]` (permanent history), `mutable_state[]` (current
status), `disposition_to_player`, `hidden_thought`, `is_protagonist`,
`first_seen_sequence`, `last_seen_sequence`, `mention_count`.

### 5.2 Per-turn update lifecycle

1. **Extract** (`worker/lib/character-codex-extractor.ts`) — gpt-4o-mini reads
   `player_input + ai_response + existing cards` and emits per-character deltas:
   new `immutable_facts`, new `mutable_state`, `retire_state` (status now false),
   `disposition`, `hidden_thought`, `is_protagonist`. Also receives the seed
   prompt (sentient) or protagonist name (GM) so it tags the protagonist
   correctly.
2. **Resolve & merge** (`characterCodexService.applyDeltas`) — match the delta to
   an existing card by name/alias (or create a new card); append to
   `immutable_facts` (keep-recent, cap **40**); reconcile `mutable_state` (drop
   retired, add new, cap **12**); update disposition/thought; `mention_count++`;
   bump `last_seen_sequence`.
3. **Supersede** (§7) — retired facts evict matching memory vectors.
4. **Compact** (§5.4) — if a card's fact list grows past 24, an async LLM pass
   distills it back down.

### 5.3 Injection ranking (recency-aware)

At dispatch, `generation.service` pulls a 40-card candidate pool then ranks with
`rankCodexForInjection` and injects the **top 16**:

```
score = mention_count × 0.5 ^ (turnsSinceLastSeen / 80)
```

(`RANK_HALF_LIFE = 80`.) The protagonist is always pinned first. This makes the
injected set track the **current** cast: a character who was central long ago but
dormant for many turns decays out, while a recently-active one rises — instead of
ranking purely by lifetime mention frequency.

### 5.4 Bounded growth (fact compaction)

Caps never silently drop *new* facts: `uniqKeepRecent` keeps the most recent on
overflow. When a card exceeds 24 immutable facts, `worker/lib/codex-compactor.ts`
(`compactImmutableFacts`, gpt-4o-mini, async) distills the list to ~16, **merging
redundancies while preserving identity and irreversible history** (deaths,
marriages, powers, betrayals). The prompt shows only the most recent 14 facts.

### 5.5 The protagonist

Every instance has exactly one locked protagonist:

- **Sentient world / character** → the AI persona, seeded deterministically from
  the template at instance creation (`characterCodexService.seedProtagonist`).
  Its identity lives in the seed prompt, so the codex injects only its
  **evolving state** ("YOUR EVOLVING STATE …"), not a redundant persona card.
- **GM world** → the **player's own character**, established by a "Who are you?"
  onboarding on first play. The codex injects a "THE PLAYER (protagonist)" block
  so the narrator addresses them consistently.

The protagonist flag is sticky and pinned to the top of the roster.

---

## 6. Memories in depth

Memories are the **semantic fact-file** — atomic, durable facts retrieved by
relevance from *anywhere* in history.

- **Creation** (`memory.processor.ts`): gpt-4o-mini distills the turn into
  memories, rates importance 1–5, embeds each (`text-embedding-3-small`, 1024
  dims), upserts into Pinecone `mem_{instanceId}` (vector id = random UUID,
  metadata carries `text`, `importance`, `mongo_id`, `created_at`), and writes
  the `MemoryDoc` to Mongo (`pinecone_id` = vector id).
- **Retrieval** (`generation.processor.ts`): the turn's query is embedded and the
  top-K most similar vectors are pulled and injected as "THINGS YOU REMEMBER
  ABOUT THIS PLAYER". RAG **queries Pinecone directly and does not filter
  `is_archived`** — a critical detail for §7.
- **Editing** (`memory.service.editMemory`): re-embeds the new text and upserts
  onto the **same `pinecone_id`**, so edits fully propagate to Pinecone.
  `deleteMemory` removes the vector.

---

## 7. Continuity: keeping facts from going stale

A long story constantly contradicts its own past ("she breaks the engagement").
Staleness is fought at **two layers**:

### 7.1 Codex reconciliation (the authoritative layer)

`mutable_state` is **not** append-only. Each turn the extractor emits
`retire_state` — existing current-status items that just became false.
`reconcileMutableState` drops the retired items (normalized fuzzy match) before
adding new ones, so the card's "current state" never holds contradictions. Since
the codex is injected as high-authority canon, an accurate card overrides drift.

### 7.2 Memory-vector supersession (the retrieval layer)

The codex card is accurate, but the **old memory vectors still exist** and RAG
can still retrieve them (it doesn't filter `is_archived`). So
`src/services/memory-supersession.service.ts` closes the gap: for each retired
fact, it embeds the fact text, finds memory vectors with similarity **≥ 0.82**
created **before** this turn, **deletes those vectors** (stops retrieval) and
archives the Mongo docs (kept for provenance). Triggered fire-and-forget from the
codex pipeline using `retire_state`, and also on a **character edit** that
removes facts (see §9).

> Threshold is deliberately high (precision over recall): a miss just leaves the
> codex to override; a false positive would erase a valid memory.

---

## 8. Scene summaries

Chapter recaps that bridge the distant past and the last few verbatim turns.
**Mongo-only — not vectorized.**

- **Trigger** (`generation.processor.ts`): `current_scene.turn_count` increments
  while the scene `tag` stays the same and resets when it changes. When a scene
  reaches `SCENE_SUMMARY_BLOCK` (12) turns, a `summarize` job is enqueued for
  that block **and the counter resets**, so blocks are **non-overlapping**
  (1–12, 13–24, …) rather than re-summarizing every turn.
- **Compress** (`summary.processor.ts`): gpt-4o-mini compresses the block's turns
  into exactly 2 self-contained paragraphs, stored with the `event_range` it
  covers.
- **Retrieve**: `generation.service` injects the latest summary whose range ends
  *before* the recent-turns window as "PREVIOUS SCENE SUMMARY" — so turns that
  scrolled out of the verbatim window are still represented, compressed.

---

## 9. Editing & propagation

Players have agency over their record, and each edit propagates correctly:

| Edit | Endpoint | Propagation |
|------|----------|-------------|
| **Event** | `PUT /chronicle/event/:id` | Re-curates: deletes memories sourced from the event (+ their vectors) and re-queues curation |
| **Memory** | `PUT /chronicle/memory/:id` | Re-embeds and upserts the same `pinecone_id` (delete removes the vector) |
| **Character / protagonist** | `PUT /chronicle/character/:id` | Updates the card; **facts the edit removes are sent to supersession** so stale memories about them can't resurface |

A character card is **per-instance and not vectorized**, so an edit is cheap and
takes effect next turn. It does **not** touch `global_lore` (the world's static,
creator-owned backdrop) — editing a character changes *your* story's canon, not
the world. Removed facts trigger supersession (§7.2); the high-authority card
also overrides anything retrieved. World **lore** itself remains creator-only.

---

## 10. How it converges in the prompt

Each turn, `prompt-builder.ts` assembles (static, cacheable prefix first):

```
[static] world identity/POV framing + WORLD LORE (global_lore) + format rules
[dynamic] tone
          protagonist block (evolving state | player identity)
          CANONICAL CHARACTER CODEX (top-16 NPC cards, recency-ranked)
          player canonical narration (this turn's asterisk actions)
          CURRENT WORLD STATE / FLAGS (omitted when stat-less)
          RELEVANT LORE DETAILS (RAG from lore_{templateId})
          THINGS YOU REMEMBER (RAG from mem_{instanceId})
[system] PREVIOUS SCENE SUMMARY
[history] recent events (sliding window, token-budgeted)
[user]    the current player message
```

Lore is the static stage; the **codex** is who the cast canonically is right now;
**memories** are what's relevant from the whole history; **summaries** bridge the
gap; **recent turns** carry the immediate flow. Every layer is bounded — which is
why the prompt size, and therefore cost per turn, stays flat no matter how long
the story runs.

---

## Key files

| Concern | File |
|---------|------|
| Prompt assembly | `src/utils/prompt-builder.ts` |
| Codex service (merge, rank, seed, edit, compact write) | `src/services/character-codex.service.ts` |
| Codex extraction | `worker/lib/character-codex-extractor.ts` |
| Codex compaction | `worker/lib/codex-compactor.ts` |
| Memory curation | `worker/processors/memory.processor.ts` |
| Memory edit/recuration | `src/services/memory.service.ts` |
| Memory supersession | `src/services/memory-supersession.service.ts` |
| Scene summaries | `worker/processors/summary.processor.ts` |
| Per-turn orchestration | `worker/processors/generation.processor.ts` |
| Codex load + ranking + summary retrieval | `src/services/generation.service.ts` |
