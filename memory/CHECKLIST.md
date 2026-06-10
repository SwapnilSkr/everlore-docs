# Infinite Memory Checklist

This checklist tracks the path from the current bounded-memory architecture to a
Supermemory-level story memory system.

## Already In Place

- [x] Durable event log in Mongo.
- [x] Chronicle pagination for player-visible history.
- [x] Bounded prompt raw-event window.
- [x] Bounded Play load window.
- [x] Active client event cap.
- [x] Active client memory cap.
- [x] Local SQLite cache pruning.
- [x] Memory extraction into Mongo + Pinecone.
- [x] Character codex with immutable facts and mutable state.
- [x] Mutable-state retirement for character continuity.
- [x] Memory-vector supersession for retired facts.
- [x] Scene summaries for older raw turns.
- [x] Summary observability logs.
- [x] Summary repair maintenance job.
- [x] Player event edit re-curates memories.
- [x] Replay/select variant re-curates memories.
- [x] Rewind removes later events/projections and recomputes state.
- [x] Memory edit re-embeds vectors.

## Phase 1: Projection Provenance

- [ ] Add projection metadata shared by memories, codex facts, summaries,
      relationships, locations, and calendar entries.
- [ ] Store `source_event_ids` on every derived projection.
- [ ] Track event revision/source revision where needed.
- [ ] Mark projections as `active`, `stale`, `superseded`, or `archived`.
- [ ] Add admin/debug endpoint to inspect projections for an event.

## Phase 2: Rich Memory Atoms

- [ ] Extend memory schema with `subject_entity_ids`.
- [ ] Extend memory schema with `object_entity_ids`.
- [ ] Add `emotional_cause`, `emotional_effect`, `relationship_delta`.
- [ ] Add `unresolved_thread` and `current_status`.
- [ ] Add `updates_memory_ids`, `extends_memory_ids`,
      `derives_from_memory_ids`.
- [ ] Update memory extraction prompt to resolve pronouns/entities explicitly.
- [ ] Update memory extraction to create emotionally-rich facts, not only
      factual summaries.
- [ ] Re-embed memories after enriched text generation.

## Phase 3: Entity Graph

- [x] Add entity collection or graph model.
      (`entities` collection + `entity-graph.service.ts`; dedup by normalized
      name/alias, created on first mention.)
- [x] Support entity types: player, protagonist, character, location, faction,
      item, quest, relationship, calendar, timeline.
      (`EntityType` = player/protagonist/character/location/faction/item/quest/
      concept; calendar + timeline deferred to Phase 5 with the TimeAnchor work.)
- [x] Add edge model with type, source events, status, and importance.
      (`entity_edges`: typed directed edges with `source_event_ids` provenance,
      status, importance, and `weight` for meter edges. The label is part of
      edge identity — each narrative assertion is its own edge, so provenance
      pruning on rewind/edit is exact and never leaves a stale label.)
- [x] Link memory atoms to entities.
      (New extractions populate `subject_entity_ids`/`object_entity_ids`;
      string `subjects`/`objects` kept for back-compat. Entity-neighborhood
      retrieval is fused into hybrid RAG when the player names an entity.)
- [x] Link codex cards to character entities.
      (1:1 `characters.entity_id` ↔ `entities.character_id`, synced every turn
      and re-linked after rewind re-mints card ids; lazy backfill for old worlds.)
- [x] Add relationship edges for trust, affection, fear, betrayal, forgiveness,
      secrecy, debt, rivalry.
      (Meter edges trust/affection/fear/rivalry are projected from the codex
      relationship ledger; free-form `relationship` edges with labels come from
      relationship-type memory atoms — betrayal/forgiveness/debt live in the
      label rather than as fixed types.)
- [x] Add graph repair job for edges pointing to stale/deleted events.
      (Inline in `rewindToSequence` — entities born in removed turns deleted,
      edge provenance pruned, empty edges dropped, meter edges re-projected —
      plus event edit/replay provenance pruning and a `repair_entity_graph`
      maintenance task; verified by `scripts/rewind-audit.ts`.)

## Phase 4: Hybrid Retrieval

- [ ] Add BM25/text index over memory text, entity names, places, promises, and
      event labels.
- [ ] Retrieve by vector similarity.
- [ ] Retrieve by exact names/keywords.
- [x] Retrieve by entity neighborhood.
      (Shipped with Phase 3: memories linked by id to entities the player
      named, fused into RRF alongside vector + keyword.)
- [ ] Retrieve by timeline/calendar filters.
- [ ] Merge/rerank retrieval results into a context packet.
- [ ] Track retrieval usage and update access counts.

## Phase 5: Calendar And Timelines

- [ ] Add world calendar definitions.
- [ ] Add `TimeAnchor` to events/projections.
- [ ] Track `sequence`, `real_time`, `story_calendar`, `event_time_label`.
- [ ] Add `timeline_id`.
- [ ] Support flashbacks without rewriting sequence order.
- [ ] Support alternate branches/time travel.
- [ ] Add calendar UI backed by story-time anchors.
- [ ] Link calendar entries to Chronicle events and summaries.

## Phase 6: Locations And Travel

- [ ] Add location entities.
- [ ] Track current player location per instance.
- [ ] Add travel events.
- [ ] Update calendar/story time during travel.
- [ ] Store location state and permanent location facts.
- [ ] Retrieve location memories when entering a place.
- [ ] Add "what happened here before?" Chronicle view.

## Phase 7: Side-Character Chats

- [ ] Add side-chat event type.
- [ ] Pin active side character in context packet.
- [ ] Scope secrets by who knows them.
- [ ] Update relationship graph from side chats.
- [ ] Let main narration retrieve side-chat memories only when canonically
      appropriate.

## Phase 8: Context Packet Builder

- [ ] Build explicit context packet before `buildPrompt`.
- [ ] Separate static canon from dynamic state.
- [ ] Include current scene, location, story time, timeline branch.
- [ ] Include relevant relationship state.
- [ ] Include emotionally-relevant memories.
- [ ] Include location/time memories.
- [ ] Include recent raw turns.
- [ ] Add token budgeting per packet section.

## Phase 9: Advanced Compaction

- [ ] Add chapter summaries over scene summaries.
- [ ] Add arc summaries over plot/relationship threads.
- [ ] Embed summaries for semantic summary retrieval.
- [ ] Add continuity audits comparing codex, memories, summaries, and graph
      state.
- [ ] Add cold archival strategy only when storage pressure is real.

## Phase 10: Product Surfaces

- [ ] Calendar view.
- [ ] Relationship ledger.
- [ ] Location journal.
- [ ] Character-specific memory view.
- [ ] "What this character remembers about you."
- [ ] Promise/quest tracker.
- [ ] Timeline branch viewer.
- [ ] Memory-aware recaps.
- [ ] Advanced Chronicle search and filters.

