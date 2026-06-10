# Core Philosophy

## The problem Everlore solves

After thousands of chat turns, the full transcript is **millions of words**. No AI can read all of it every turn — it would be slow, expensive, and confusing.

But the player should still feel like the world **remembers** who they met, what they promised, how relationships changed, where they've been, and when things happened.

---

## The big idea (in one diagram)

```text
┌─────────────────────────────────────────────────────────────┐
│  FULL STORY (durable, player-visible)                        │
│  • Every turn saved in Mongo as an "event"                   │
│  • Chronicle / Lore Tome shows as much as you want           │
│  • Rewind can roll back to any turn                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │  projections (derived, rebuildable)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  AI BRIEFING (small, fixed-ish size each turn)               │
│  • Last ~6 turns (token-budgeted)                            │
│  • ~25 retrieved memories                                    │
│  • ~16 character cards                                       │
│  • 1 scene summary + distant chapter recaps                  │
│  • Current place, story date, open promises                  │
└─────────────────────────────────────────────────────────────┘
```

**Rule:** Events are the source of truth. Everything else — memories, character cards, summaries, relationship edges, place facts — is a **projection** that can be rebuilt or pruned when events change.

---

## Design principles actually used in code

### 1. Bounded prompt, unbounded history

Turn 50 and turn 13,000 build roughly the **same size** prompt. History depth does not inflate what the model sees.

Caps live in `prompt-builder.ts` and `context-packet.service.ts`:
- Recent turns: 6 (plus a hard 1000-token floor for continuity)
- Character codex: 16 cards (from a pool of 40, recency-ranked)
- Memories: ~25 (hybrid search)
- Lore: ~10
- Open threads: 5
- Token budgets per section so one huge codex can't squeeze out recent conversation

### 2. Projections must survive edits and rewinds

When you **edit** a turn, **replay** a variant, or **rewind**:
- Memories sourced from removed/changed events are deleted or re-built
- Character codex replays from **ledgered deltas** stored on each event (exact on rewind)
- Entity graph, location facts, summaries, and vectors are pruned or rebuilt
- Session cache is busted so the next turn sees fresh state

See `memory.service.ts` → `rewindToSequence`, `recurateMemoriesForEvent`.

### 3. Two filing systems, one story

From the same turn, two parallel extractors run:

| System | What it stores | How the AI uses it |
|--------|----------------|-------------------|
| **Character codex** | Structured cards: facts, current state, trust/fear meters | Injected as canon rules ("never contradict these") |
| **Memories** | Searchable fact atoms in Mongo + Pinecone | Retrieved by relevance to your current message |

They are **not** directly linked in the database — they connect through **shared names** and now **entity IDs**. Hybrid search finds memories; codex ranking + pinning finds character sheets.

### 4. Retrieval before codex selection

Older design ranked codex first, then searched memories. **Now** (Phase 8): the worker runs RAG **first**, then pins character cards for:
- Names you typed this turn
- Characters the retrieved memories are about

This fixes "ask about a dormant character" — memories come back via search; their card gets pinned even if they fell out of the top-16 recency list.

### 5. Main story vs private side chats

Side-character private chats are **real events** in the same ledger (same sequence counter, rewind-safe) but:
- **Excluded** from main story timeline, summaries, Play feed, recaps
- **Memories** from side chats are tagged with who knows them
- Main narration **cannot** see a private secret unless the protagonist is a knower (fail-closed)

This is a hard privacy invariant — grep for `side_chat` exclusion when adding new reads.

### 6. Rules-first prose cleanup

After the AI writes, `prose-hygiene.ts` checks formatting (quotes, italics, POV). LLM repair only runs if rules fail. A voice reminder is injected right before your turn so weak models don't copy bad patterns from old text.

### 7. Cost per turn is mostly flat

Each turn pays for:
- One narration LLM call (streamed)
- One metadata/scene extraction call
- One codex extraction call
- One memory curation job (async)
- Embeddings for new memories

That cost **does not** grow with total turns played. Storage grows; prompt size does not.

---

## What "infinite memory" means here

**Yes:** Facts, emotions, promises, relationship shifts, and place history can persist across 10,000+ turns via search + summaries + codex.

**Partially:** Story calendar, timeline branches, travel, and location state work — but full "graph reasoning" (NPC-to-NPC relationships, complex timeline UI) is simpler than the original vision doc.

**Not yet:** Explicit memory-to-memory version links (`updates` / `extends` / `derives`); broader keyword search beyond memory text.

---

## Mental model for gameplay

Think of Everlore as:

1. **A diary** (events) — everything that happened, in order
2. **A filing cabinet** (memories) — searchable index cards pulled out when relevant
3. **Character sheets** (codex) — RPG-style stats and facts for the cast
4. **Chapter notes** (summaries) — compressed recaps of old scenes
5. **A map + calendar** (entities, locations, time anchors) — where and when
6. **A briefing packet** (context packet) — what the narrator reads before answering

The app is both the **game** (Play) and the **library** (Lore Tome) over that world.
