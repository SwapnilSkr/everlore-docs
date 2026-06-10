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

- [x] Add projection metadata shared by memories, codex facts, summaries,
      relationships, locations, and calendar entries.
      (`projection.model.ts`: shared `ProjectionStatus` + `ProjectionProvenance`
      used by memories, scene summaries, and entity edges; codex deltas are
      ledgered on their source event. Calendar entries arrive in Phase 5.)
- [x] Store `source_event_ids` on every derived projection.
      (Memories ✓, entity edges ✓, codex deltas live ON the event ✓; scene
      summaries trace by `event_range` instead — equivalent for staleness.)
- [ ] Track event revision/source revision where needed.
      (Deferred: events keep full `edit_history`, and edits/replays already
      re-curate or stale every covering projection, so per-projection revision
      numbers have no consumer yet.)
- [x] Mark projections as `active`, `stale`, `superseded`, or `archived`.
      (Memories: 'active' on create, 'superseded' by supersession/dedup,
      'archived' by importance decay — `is_archived` stays the retrieval gate
      for back-compat. Summaries: active/stale. Edges: active/stale/archived.)
- [x] Add admin/debug endpoint to inspect projections for an event.
      (`GET /admin/events/:eventId/projections` — memories with effective
      status, entity edges, covering summaries, codex deltas, linked entities.)

## Phase 2: Rich Memory Atoms

- [x] Extend memory schema with `subject_entity_ids`.
      (Shipped with Phase 3: extraction resolves subjects to entity ids;
      string `subjects` kept for back-compat.)
- [x] Extend memory schema with `object_entity_ids`.
- [x] Add `emotional_cause`, `emotional_effect`, `relationship_delta`.
      (Plus `emotional_valence`; extraction prompt asks for cause AND lasting
      effect explicitly.)
- [x] Add `unresolved_thread` and `current_status`.
      (`unresolved_thread` + `resolved_at` power the open-threads prompt
      section; lifecycle is the shared `status` from projection provenance.)
- [ ] Add `updates_memory_ids`, `extends_memory_ids`,
      `derives_from_memory_ids`.
      (Not built: no stored memory→memory version links yet. Supersession
      works at the vector level (retired facts evict stale vectors) and dedup
      merges near-duplicates, but neither records WHICH memory replaced which.)
- [x] Update memory extraction prompt to resolve pronouns/entities explicitly.
      (Extraction is grounded in the codex roster; atoms must be
      self-contained with explicit names.)
- [x] Update memory extraction to create emotionally-rich facts, not only
      factual summaries.
- [x] Re-embed memories after enriched text generation.
      (Moot as a separate step: extraction emits enriched text directly and
      embeds it once; player edits re-embed onto the same vector id.)

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
- [x] Retrieve by vector similarity.
      (Pinecone memory/lore search remains the semantic arm.)
- [x] Retrieve by exact names/keywords.
      (Mongo text search over memory text + subjects/objects; broader entity
      names/event-label BM25 remains a low-priority expansion of the index.)
- [x] Retrieve by entity neighborhood.
      (Shipped with Phase 3: memories linked by id to entities the player
      named, fused into RRF alongside vector + keyword.)
- [x] Retrieve by timeline/calendar filters.
      (Memories now carry `time_anchor` + `timeline_id`; RAG scopes memory
      retrieval to the active timeline plus parent branches up to their fork
      sequence, while legacy rows are treated as main-branch-compatible only
      when main is in the ancestry. Calendar-date range filtering can build on
      the indexed fields.)
- [x] Merge/rerank retrieval results into a context packet.
      (RRF fusion of vector + keyword + entity arms feeds the Phase 8
      ContextPacket; timeline-filter reranking remains for Phase 5.)
- [x] Track retrieval usage and update access counts.
      (`queryRag` bumps `access_count` / `last_accessed_at`; importance decay
      consumes those fields.)

## Phase 5: Calendar And Timelines

- [x] Add world calendar definitions.
      (`story_calendars` collection; every instance gets a default fantasy
      calendar with eras/months/weekdays, ready for custom calendar authoring.)
- [x] Add `TimeAnchor` to events/projections.
      (`events.time_anchor`, `memories.time_anchor`, and
      `world_instances.current_time_anchor`; reset/rewind repair the cursor.)
- [x] Track `sequence`, `real_time`, `story_calendar`, `event_time_label`.
      (`TimeAnchor` stores all four. Calendar ticks advance the day-level story
      date deterministically and preserve labels such as "several days".)
- [x] Add `timeline_id`.
      (Events, memories, instances, and `timeline_branches` use branch ids;
      default branch is `main`.)
- [x] Support flashbacks without rewriting sequence order.
      (`PUT /chronicle/calendar/event/:eventId/time-anchor` changes an event's
      story date/timeline anchor while leaving `sequence` untouched, and
      propagates the anchor to sourced memories.)
- [x] Support alternate branches/time travel.
      (`timeline_branches` plus Chronicle endpoints to fork/switch active
      timelines; new events inherit the active branch.)
- [x] Add calendar UI backed by story-time anchors.
      (Backend surface shipped: `GET /chronicle/calendar/:instanceId` returns
      calendars, timelines, current anchor, and Chronicle-linked events. A
      dedicated frontend view can now consume it.)
- [x] Link calendar entries to Chronicle events and summaries.
      (Chronicle events carry `time_anchor`; calendar endpoint returns event ids
      + sequences. Summaries remain linked by event ranges.)

## Phase 6: Locations And Travel

- [x] Add location entities.
      (Location entity type + memory extraction mint linked location entities;
      gameplay/location-state systems below remain unbuilt.)
- [x] Track current player location per instance.
      (`world_instances.current_location` stores the current location entity
      anchor; generation updates it from metadata when a turn establishes a
      concrete end-of-scene place, otherwise it carries the prior place.)
- [ ] Add travel events.
- [ ] Update calendar/story time during travel.
- [ ] Store location state and permanent location facts.
- [x] Retrieve location memories when entering a place.
      (Events and memories now carry `location_anchor`; RAG fuses a current
      location memory arm with vector/keyword/entity/timeline retrieval.)
- [x] Add "what happened here before?" Chronicle view.
      (Shipped — server commit `de747cd`, app commit `733d598`. Per-place
      journal screen: events + memories anchored to a location via
      `GET /chronicle/locations/:instanceId/:locationEntityId`. Reached from the
      Phase 10 Location Journal "Places" tab.)

## Phase 7: Side-Character Chats

- [ ] Add side-chat event type.
- [ ] Pin active side character in context packet.
- [ ] Scope secrets by who knows them.
- [ ] Update relationship graph from side chats.
- [ ] Let main narration retrieve side-chat memories only when canonically
      appropriate.

## Phase 8: Context Packet Builder

- [x] Build explicit context packet before `buildPrompt`.
      (`context-packet.service.ts`, built in the WORKER so retrieval runs
      BEFORE codex selection: cards pin for direct name mentions AND for
      characters the retrieved memories are about (memory-driven pinning via
      `retrievedEntityIds`). Dispatch is now thin: session + consent + enqueue.)
- [x] Separate static canon from dynamic state.
      (Prompt builder: byte-stable cacheable static prefix — identity, voice,
      lore, format rules — then per-turn dynamic sections.)
- [x] Include current scene, location, story time, timeline branch.
      (Story time + timeline branch now inject through the context packet;
      current location is injected as a place anchor. Current scene remains
      covered by recents + scene summary.)
- [x] Include relevant relationship state.
      (Codex cards carry relationship meters, disposition, hidden thoughts;
      pinning ensures the cards retrieval implicates ride along.)
- [x] Include emotionally-relevant memories.
      (Rich atoms + hybrid retrieval + open-threads section.)
- [x] Include location/time memories.
      (Timeline-scoped memories now work through `time_anchor`; location-state
      memories now retrieve through `location_anchor`. Rich independent
      location-state entries remain Phase 6 follow-up work.)
- [x] Include recent raw turns.
- [x] Add token budgeting per packet section.
      (Packet-level allocator: reference sections share the pool left after
      the static prefix, proportionally and capped; recent-turn continuity has
      a HARD 1000-token floor that survives even oversized prefixes.)

## Phase 9: Advanced Compaction

- [ ] Add chapter summaries over scene summaries.
- [ ] Add arc summaries over plot/relationship threads.
- [ ] Embed summaries for semantic summary retrieval.
- [ ] Add continuity audits comparing codex, memories, summaries, and graph
      state.
- [ ] Add cold archival strategy only when storage pressure is real.

## Phase 10: Product Surfaces

- [x] Calendar view.
      (Almanac tab in the Lore Tome — app commit `5079600`. Renders the current
      story-time cursor through the world calendar plus events grouped by
      in-world date with milestone/time-jump markers. Backed by the existing
      `GET /chronicle/calendar/:instanceId`.)
- [x] Relationship ledger.
      (Bonds tab in the Lore Tome — server commit `4e23bf4`, app commit
      `88d6110`. Per-character trust/affection/fear/rivalry meters +
      disposition from the codex ledger, plus the narrative `relationship`
      edges that shifted each bond, via `GET /chronicle/relationships/
      :instanceId`. `hidden_thought` stays private.)
- [x] Location journal.
      (Places tab in the Lore Tome — server commit `de747cd`, app commit
      `733d598`. Lists every visited place (current location pinned first, with
      moment/echo counts) via `GET /chronicle/locations/:instanceId`; tapping a
      place opens the per-place "what happened here before?" journal.)
- [ ] Character-specific memory view.
- [ ] "What this character remembers about you."
- [ ] Promise/quest tracker.
- [x] Timeline branch viewer.
      (Almanac "Realities" switcher — app commit `5079600`. Lists timeline
      branches, highlights the active one, and switches the active reality via
      `PUT /chronicle/calendar/:instanceId/timeline/active`.)
- [ ] Memory-aware recaps.
- [ ] Advanced Chronicle search and filters.
