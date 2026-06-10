# Memory Layers Explained

Each layer of memory — what it stores, where it lives, and how it connects.

---

## Layer stack

```text
                    ┌──────────────┐
                    │   EVENTS     │  ← source of truth (every turn)
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │ MEMORIES   │  │   CODEX    │  │ SUMMARIES  │
    │ Mongo+Vec  │  │ characters │  │ scene/ch/arc│
    └─────┬──────┘  └─────┬──────┘  └────────────┘
          │               │
          └───────┬───────┘
                  ▼
           ┌────────────┐
           │  ENTITIES  │  ← WHO / WHERE as stable IDs
           │  + EDGES   │
           └────────────┘
```

---

## 1. Events (the diary)

**Collection:** `events`  
**Model:** `world-event.model.ts`

Every turn is one row:

| Field | Meaning |
|-------|---------|
| `sequence` | Turn number (never reused) |
| `type` | `narration`, `calendar_tick`, `travel`, `intimate`, `side_chat` |
| `data.player_input` / `ai_response` | What was said |
| `data.codex_deltas` | Character changes this turn (for rewind replay) |
| `data.state_mutations` / `flag_mutations` | Game stats |
| `data.travel` | `{ from, to }` when you moved places |
| `time_anchor` | Story calendar position |
| `location_anchor` | Where this turn happened |

**Player-visible:** Chronicle Timeline tab, Play bubbles (last 20 loaded, link to older)

**AI-visible:** Last 6 turns only (plus summaries for older)

---

## 2. Memories (searchable fact cards)

**Collection:** `memories`  
**Vectors:** Pinecone namespace `mem_{instanceId}`  
**Model:** `memory.model.ts`

Each memory is one atomic fact, e.g. *"Mira forgave you after the bridge betrayal but trust remains fragile."*

| Field | Purpose |
|-------|---------|
| `text` | The fact (embedded for search) |
| `importance` | 1–5 ranking |
| `source_event_ids` | Which turn(s) created it |
| `subjects` / `objects` | Name strings (legacy + display) |
| `subject_entity_ids` / `object_entity_ids` | Linked entities |
| `emotional_*`, `relationship_delta` | Rich emotional structure |
| `unresolved_thread` | Open promise/conflict flag |
| `time_anchor`, `timeline_id` | When / which reality |
| `location_anchor` | Where it happened |
| `origin`, `known_by_entity_ids` | Side-chat privacy |

**How retrieved:** Hybrid RAG — meaning similarity + keyword match + entity neighborhood + current place + timeline filter.

**Player-visible:** Echoes tab (search/filter/edit), in-play tap on names, character memory screens

**Caps:** ~25 in prompt; app keeps last 50 live; importance decay archives stale low-importance memories after 30 days unused

---

## 3. Character codex (cast sheets)

**Collection:** `characters`  
**Model:** `character-profile.model.ts`  
**Service:** `character-codex.service.ts`

Structured RPG-style cards:

| Field | Purpose |
|-------|---------|
| `immutable_facts` | Permanent history (capped, compacted async) |
| `mutable_state` | Current status ("exiled north", "engaged") |
| `relationship` | Trust / affection / fear / rivalry meters (0–100) |
| `disposition_to_player`, `hidden_thought` | Narrative stance |
| `entity_id` | Link to entity graph |
| `mention_count`, `last_seen_sequence` | Recency ranking |

**Updated:** Every turn via LLM extraction → `applyDeltas` → ledger stored on event

**Injected:** Top 16 by recency score, plus **pinned** if you named them or memories implicate them

**Player-visible:** Bonds tab, Thoughts sheet, bond rail on Play

---

## 4. Summaries (compressed history)

Three tiers — all rebuildable projections:

| Tier | Trigger | Collection |
|------|---------|------------|
| **Scene** | Every 12 turns same scene tag | `scene_summaries` |
| **Chapter** | Every 8 scene summaries | `chapter_summaries` |
| **Arc** | Every 4 chapters | `arc_summaries` |

**Vectors:** Pinecone `sum_{instanceId}` for semantic "what past chapter is relevant now?"

**Player-visible:** Recap spine (latest scene summary), indirectly via better long-range AI recall

**Prompt section:** "RELEVANT PAST CHAPTERS" when vector search finds distant matches

---

## 5. Entity graph (nouns with IDs)

**Collections:** `entities`, `entity_edges`  
**Service:** `entity-graph.service.ts`

Turns loose names into stable things:

| Entity type | Example |
|-------------|---------|
| `player` | The human |
| `protagonist` | Main character (sentient AI or GM player character) |
| `character` | NPC codex entry |
| `location` | Named place |
| `faction`, `item`, `quest`, `concept` | Extensible |

**Edges** connect entities:
- Meter edges: trust/affection/fear/rivalry (from codex)
- Narrative edges: "betrayed", "owes debt" (from relationship memories)
- Each edge tracks `source_event_ids` for rewind pruning

**Location entities** also carry:
- `location_facts` — permanent canon ("the well was poisoned")
- `location_state` — current condition ("market burned")

Event-sourced: facts prune when their source turn is rewound or edited.

---

## 6. Time & timelines

**Service:** `time.service.ts`  
**Collections:** `story_calendars`, `timeline_branches`

Each instance has:
- A **fantasy calendar** (eras, months, weekdays)
- A **current story date** on the instance
- A **timeline branch** (default `main`; can fork for alternate realities)

Events and memories carry `time_anchor` so RAG can scope to the active timeline + parent branches.

**Flashbacks:** Admin/chronicle can re-anchor an event's story date without changing sequence order.

**Player-visible:** Almanac tab (calendar, realities switcher, dated events)

---

## 7. Locations & travel

**Service:** `location.service.ts`  
**Cursor:** `world_instances.current_location`

- **Travel events** created when end-of-turn place ≠ previous place (server-detected)
- **Time advances** when narration says "weeks passed" (not only explicit wait button)
- **Place journal:** events + memories + facts for one location

**Player-visible:** Places tab, travel markers in Timeline/Almanac, location journal drill-down

---

## How layers connect (honest picture)

```text
Same turn
   │
   ├──► Memory atoms (searchable, emotional)
   ├──► Codex deltas (structured character state)
   ├──► Location facts (if place changed)
   └──► Entity edges (relationship shifts)

Connections:
   • entity_id links codex card ↔ entity
   • memory subject_entity_ids link atoms ↔ entities
   • source_event_ids link everything ↔ events
   • Names in text are the human-readable glue
```

There is still **no** full memory-version graph (`updates` / `extends` / `derives` between memories) — supersession works at vector/text level today.

---

## Privacy: two visibility worlds

| Surface | Events | Memories |
|---------|--------|----------|
| **Main story** (Play, Timeline, Recap, Echoes, Threads, Places, Calendar) | Excludes `side_chat` | Protagonist must be in `known_by` for side-chat origin |
| **Side chat thread** | Only that character's side_chat events | Scoped to that conversation |

Fail-closed: if unsure, private memories don't leak to main narration.
