# All Ten Phases — What Shipped

Each checklist phase translated into plain English and where it lives in code.

**Legend:** ✅ Done · 🔄 Reopened for next planning · ⏸ Deferred by design

---

## Already in place (pre-phase foundation)

These existed before the big memory push:

- Durable event log, Chronicle pagination
- Bounded prompt window (6 turns) and Play load (20 turns)
- Client caps: 100 events, 50 memories in active state; SQLite cache prunes at 200
- Memory extraction → Mongo + Pinecone
- Character codex with facts + mutable state
- Scene summaries, rewind, memory edit re-embed

---

## Phase 1 — Projection provenance ✅

**Idea:** Every derived thing should know *which turn* created it and whether it's still valid.

**Shipped:**
- Shared `ProjectionStatus`: active, stale, superseded, archived
- Memories track status; summaries can go stale on edit
- Entity edges carry `source_event_ids`
- Codex deltas live **on the event** (the ledger)
- Admin endpoint: `GET /admin/events/:eventId/projections`

**Files:** `projection.model.ts`, `admin.service.ts`, `memory.model.ts`

**⏸ Deferred:** Per-projection revision counters (no consumer yet)

---

## Phase 2 — Rich memory atoms ✅ (mostly) / 🔄 (version links)

**Idea:** Memories should carry emotional weight and link to entities, not just be flat sentences.

**Shipped:**
- `subject_entity_ids`, `object_entity_ids` (plus string subjects/objects for back-compat)
- `emotional_cause`, `emotional_effect`, `relationship_delta`, `unresolved_thread`
- Extraction prompt asks for self-contained, emotionally rich atoms
- Re-embed on player edit

**Files:** `memory.model.ts`, `memory.processor.ts`

**🔄 Reopened:** `updates_memory_ids`, `extends_memory_ids`, `derives_from_memory_ids` — explicit memory evolution graph for explainability (supersession still works without these)

---

## Phase 3 — Entity graph ✅

**Idea:** Names become stable IDs; relationships become edges you can query and repair.

**Shipped:**
- `entities` collection + `entity_edges`
- Types: player, protagonist, character, location, faction, item, quest, concept
- 1:1 link: `characters.entity_id` ↔ character entity
- Memories link via subject/object entity IDs
- Meter edges (trust/affection/fear/rivalry) projected from codex
- Narrative edges from relationship memories
- Location facts on location entities (event-sourced)
- Rewind/edit graph repair inline + `repair_entity_graph` maintenance job
- Entity neighborhood fused into RAG

**Files:** `entity-graph.service.ts`, `entity.model.ts`, `entity-edge.model.ts`

---

## Phase 4 — Hybrid retrieval ✅ (mostly) / 🔄 (broader BM25)

**Idea:** Find memories by meaning AND by exact names; merge intelligently.

**Shipped:**
- Vector search (Pinecone) for lore + memories
- Mongo `$text` keyword search on memory text/subjects/objects
- Entity neighborhood retrieval
- Location-scoped retrieval
- Timeline branch filtering
- Open threads (separate section)
- RRF fusion + importance boost
- Access count tracking for decay
- Summary semantic search (`querySummaries`)

**Files:** `rag.provider.ts`, `context-packet.service.ts`

**🔄 Reopened:** Index entity names, places, promises, event labels beyond current memory text index — measure before expanding prompt surface

---

## Phase 5 — Calendar & timelines ✅

**Idea:** Story has in-world dates and alternate realities, not just turn numbers.

**Shipped:**
- `story_calendars` with eras/months/weekdays
- `TimeAnchor` on events, memories, instance cursor
- `timeline_branches` with fork/switch APIs
- Flashback: re-anchor event story date without changing sequence
- RAG scopes memories to active timeline + parent ancestry
- Calendar API for Almanac UI

**Files:** `time.service.ts`, `time.model.ts`, `generation.processor.ts`

---

## Phase 6 — Locations & travel ✅

**Idea:** Places are first-class; travel moves you and advances time when narrated.

**Shipped:**
- Location entities minted from extraction
- `current_location` on instance
- Travel events when end-of-turn place ≠ previous (server-detected)
- Narrated time skips advance calendar (not only wait button)
- Location facts + mutable location state (rewind/edit safe)
- Location memory arm in RAG
- Place journal API + UI

**Files:** `location.service.ts`, `entity-graph.service.ts` (`applyLocationFacts`), `generation.processor.ts`, `metadata-extractor.ts`

**Honest gap:** LLM extraction paths for travel/time/location fields are code-correct but not all verified on a live generated turn in production.

---

## Phase 7 — Side-character chats ✅

**Idea:** Private NPC conversations that don't pollute main story but still update bonds.

**Shipped:**
- `side_chat` event type in same ledger
- Separate worker path + context packet scoped to one character
- Secret scoping: `origin: 'side_chat'`, `known_by_entity_ids`
- Main RAG fail-closed gate
- Codex deltas from side chats (active character only)
- Excluded from main timeline, summaries, Play feed, recaps
- WS + REST thread APIs
- App: Bonds → private chat screen

**Files:** `side-chat.processor.ts`, `side-chat.service.ts`, `context-packet.service.ts` (`buildSideChatPacket`), `rag.provider.ts`

---

## Phase 8 — Context packet builder ✅

**Idea:** One explicit assembly step; retrieval before codex; token-safe sections.

**Shipped:**
- `buildContextPacket()` in worker (not dispatch)
- Memory-driven codex pinning (indirect mentions)
- Static vs dynamic prompt split (cache-friendly prefix)
- Injects: time, place, timeline, relationships, memories, threads, summaries, recents
- Per-section token budgets + proportional allocator
- Hard 1000-token floor for recent turns

**Files:** `context-packet.service.ts`, `prompt-builder.ts`

---

## Phase 9 — Advanced compaction ✅

**Idea:** Compress old history in tiers; detect drift; don't delete events.

**Shipped:**
- Scene (12 turns) → Chapter (8 scenes) → Arc (4 chapters)
- Summaries embedded in Pinecone; semantic retrieval into prompt
- "RELEVANT PAST CHAPTERS" prompt section
- Continuity audit (8 checks) + daily drift scheduler
- Admin drift status listing

**Files:** `summary.processor.ts`, `continuity-audit.service.ts`, `maintenance.processor.ts`

**⏸ Deferred:** Cold archival of old events until real storage pressure

---

## Phase 10 — Product surfaces ✅

**Idea:** Give players UI to explore the memory system.

**Shipped (Lore Tome — 7 tabs):**

| Tab | Phase 10 item |
|-----|---------------|
| Recap | Memory-aware recaps |
| Timeline | Chronicle + travel markers |
| Echoes | Advanced search/filters |
| Almanac | Calendar + timeline branch viewer |
| Places | Location journal |
| Bonds | Relationship ledger + character memory + side chat entry |
| Threads | Promise/quest tracker |

**Play surfaces:**
- Bond rail, choice chips, time skip, story timeline sheet
- Tappable names/places in prose
- Milestones, gamified meters

**App + server commits** listed in `memory/CHECKLIST.md` per item.

---

## Phase completion snapshot

```text
Phase 1  ████████████████████  Done (revision counters deferred)
Phase 2  ██████████████████░░  Done except memory-version links (reopened)
Phase 3  ████████████████████  Done
Phase 4  ██████████████████░░  Done except broader BM25 (reopened)
Phase 5  ████████████████████  Done
Phase 6  ████████████████████  Done (live-turn verification pending)
Phase 7  ████████████████████  Done (live side-chat verification pending)
Phase 8  ████████████████████  Done
Phase 9  ████████████████████  Done (cold archive deferred)
Phase 10 ████████████████████  Done
```

---

## Major commit timeline (server, newest first)

Rough order of the big memory build (~40 commits):

| Commits | Theme |
|---------|-------|
| `076a365`–`c56b9c3` | Drift audit admin + scheduler |
| `2e4daa9`–`723d366` | Travel, location facts, calendar time |
| `bf9ff3e`–`e07a6b6` | Side chats + privacy fixes |
| `541a458`–`b23cf21` | Continuity audit, arcs, chapters, summary prompt |
| `b640134`–`de747cd` | Chronicle product APIs |
| `e114341`–`be56a16` | Time, location cursor, context packet |
| `20bc34c` | Entity graph |
| `2bfabfd`–`1dbc019` | Pinning, token budgets, rewind perf |
| `495cfaa`–`1c86b04` | Codex ledger, hybrid RAG, bonds gamification |

Frontend mirrors these in Bonds, Places, Almanac, Recap, Echoes, side chat, travel markers.
