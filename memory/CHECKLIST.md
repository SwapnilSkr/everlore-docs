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

- [ ] Add entity collection or graph model.
- [ ] Support entity types: player, protagonist, character, location, faction,
      item, quest, relationship, calendar, timeline.
- [ ] Add edge model with type, source events, status, and importance.
- [ ] Link memory atoms to entities.
- [ ] Link codex cards to character entities.
- [ ] Add relationship edges for trust, affection, fear, betrayal, forgiveness,
      secrecy, debt, rivalry.
- [ ] Add graph repair job for edges pointing to stale/deleted events.

## Phase 4: Hybrid Retrieval

- [ ] Add BM25/text index over memory text, entity names, places, promises, and
      event labels.
- [ ] Retrieve by vector similarity.
- [ ] Retrieve by exact names/keywords.
- [ ] Retrieve by entity neighborhood.
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

