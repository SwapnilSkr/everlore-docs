# Everlore Server — Data Model

MongoDB collections and how they relate. Types live in `everlore-server/src/models/`.

> **IDs:** All primary keys are MongoDB **`ObjectId`** (24-char hex). There are no `usr_` / `tpl_` string prefixes in the current codebase.

For projection lifecycle rules, see [../memory/PROJECTION_AND_MUTATION_MODEL.md](../memory/PROJECTION_AND_MUTATION_MODEL.md).

---

## Architecture diagram

```text
world_templates ──creates──► world_instances ◄──player── users
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
                 events         characters       entities
                    │               │               │
                    ├──────► memories ◄──── entity_edges
                    │
                    ├──► scene_summaries ──► chapter_summaries ──► arc_summaries
                    │
                    └──► (side_chat events in same collection)

Pinecone (per instance/template):
  lore_{templateId}   mem_{instanceId}   sum_{instanceId}
```

---

## Core collections

### `users`

Account, tier, preferences (`nsfw_enabled`, narration length, etc.), auth providers (password, Google, phone).

### `world_templates`

Authored world/character blueprint: `seed_prompt`, `global_lore`, stats/flags definitions, `is_sentient`, model preferences, protagonist template, cover image URLs.

### `world_instances`

One playthrough. Key fields:

| Field | Purpose |
|-------|---------|
| `world_state`, `active_flags` | RPG stats (optional) |
| `current_scene` | `{ tag, turn_count }` for summary triggers |
| `current_time_anchor` | Story calendar cursor |
| `current_location` | `{ entity_id, name }` place cursor |
| `active_timeline_id` | Which reality branch is live |
| `meta` | Milestones, continuity audit snapshot, totals |
| Session settings | POV, chat mode, reply length, focus character, persona snapshot |

---

### `events` — source of truth

Every turn (main story, time skip, travel, side chat).

| Field | Purpose |
|-------|---------|
| `sequence` | Monotonic turn number |
| `type` | `narration`, `calendar_tick`, `travel`, `intimate`, `side_chat` |
| `data.player_input`, `data.ai_response` | Prose |
| `data.codex_deltas` | Ledger for exact codex rewind |
| `data.state_mutations`, `data.flag_mutations` | Game state |
| `data.travel` | `{ from, to }` on travel turns |
| `data.time_advanced` | Narrated time skip amount |
| `data.replay_variants` | Alternative AI responses |
| `time_anchor`, `location_anchor` | When / where |
| `side_chat` | Character ref on private chat events |

---

### `memories` — searchable fact atoms

| Field | Purpose |
|-------|---------|
| `text`, `type`, `importance` | Content + ranking |
| `pinecone_id` | Vector in `mem_{instanceId}` |
| `source_event_ids` | Provenance |
| `status` / `is_archived` | Projection lifecycle |
| `subjects`, `objects` | Name strings (display + keyword search) |
| `subject_entity_ids`, `object_entity_ids` | Graph links |
| `emotional_*`, `relationship_delta` | Rich atoms |
| `unresolved_thread`, `resolved_at` | Open promises |
| `time_anchor`, `timeline_id` | Story time / branch |
| `location_anchor` | Place link |
| `origin`, `known_by_entity_ids` | Side-chat privacy |

---

### `characters` — codex cards

| Field | Purpose |
|-------|---------|
| `canonical_name`, `aliases`, `entity_id` | Identity + graph link |
| `immutable_facts`, `mutable_state` | History + current status |
| `relationship` | Trust/affection/fear/rivalry meters |
| `disposition_to_player`, `hidden_thought` | Narrative stance |
| `is_protagonist` | Locked main character |
| `mention_count`, `last_seen_sequence` | Recency ranking |

---

### `entities` — graph nodes

Types: `player`, `protagonist`, `character`, `location`, `faction`, `item`, `quest`, `concept`.

Location entities also carry:
- `location_facts[]` — permanent canon (event-sourced)
- `location_state[]` — mutable condition (event-sourced)

### `entity_edges` — graph relationships

Typed edges (`trust`, `affection`, `fear`, `rivalry`, `relationship`, …) with `source_event_ids`, `weight`, `status`.

---

## Summary tiers (projections)

| Collection | Trigger | Key fields |
|------------|---------|------------|
| `scene_summaries` | Every 12 turns same scene tag | `summary_text`, `event_range`, `status` |
| `chapter_summaries` | Every 8 scene summaries | Same pattern |
| `arc_summaries` | Every 4 chapters | Same pattern |

Embedded in Pinecone `sum_{instanceId}` with deterministic ids `{tier}_{start}_{end}`.

---

## Time & calendar

| Collection | Purpose |
|------------|---------|
| `story_calendars` | Era/month/weekday definitions per instance |
| `timeline_branches` | Named realities (`main` + forks) |

---

## Supporting collections

| Collection | Purpose |
|------------|---------|
| `personas` | Reusable player identity profiles |
| `nsfw_lexicon` | Terms for NSFW routing classifier |
| `generation_logs` | Per-turn observability (non-blocking writes) |
| `dead_letter_jobs` | Failed generation jobs for inspection |

---

## Pinecone namespaces

| Namespace | Contents |
|-----------|----------|
| `lore_{templateId}` | Template `global_lore` chunks |
| `mem_{instanceId}` | Memory embeddings |
| `sum_{instanceId}` | Summary embeddings |

Metadata on memory vectors includes `mongo_id`, `importance`, and side-chat scope fields for fail-closed RAG.

---

## Indexes

Defined in `src/config/mongo-indexes.ts`, reconciled on connect. Notable:

- Text index on memories (text, subjects, objects) for keyword RAG + Echoes search
- Timeline / location / entity id fields on memories and events
- Summary range indexes for stale detection on edit/rewind

---

## Privacy invariant (data reads)

**Main-story surfaces** must:
1. Filter events: `type: { $ne: 'side_chat' }`
2. Filter memories: protagonist ∈ `known_by_entity_ids` OR `origin !== 'side_chat'`

See `memory.service.mainVisibleMemoryScope()` and `rag.provider.ts`.
