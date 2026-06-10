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
- [~] (IN PROGRESS) Add travel events.
      (Extraction-driven: scene-metadata extractor emits travel{from,to};
      a turn that moves the protagonist between concrete places becomes a
      `travel` event.)
- [~] (IN PROGRESS) Update calendar/story time during travel.
      (Extractor emits in-world time elapsed; narrated time skips — travel or
      otherwise — now advance the day-level calendar, not just explicit waits.)
- [~] (IN PROGRESS) Store location state and permanent location facts.
      (Location entities gain provenance-tracked current_state + permanent_facts,
      applied from extraction, pruned on rewind/edit, surfaced in the journal.)
- [x] Retrieve location memories when entering a place.
      (Events and memories now carry `location_anchor`; RAG fuses a current
      location memory arm with vector/keyword/entity/timeline retrieval.)
- [x] Add "what happened here before?" Chronicle view.
      (Shipped — server commit `de747cd`, app commit `733d598`. Per-place
      journal screen: events + memories anchored to a location via
      `GET /chronicle/locations/:instanceId/:locationEntityId`. Reached from the
      Phase 10 Location Journal "Places" tab.)

## Phase 7: Side-Character Chats

- [x] Add side-chat event type.
      (Server commit `e07a6b6`. `type: 'side_chat'` in the SAME ledger +
      sequence counter (rewind/time anchors/continuity audit exact by
      construction) with a `side_chat` character anchor. Worker path streams an
      in-character reply from the codex card; story time/scene/location cursors
      untouched. Excluded from main recents, scene summaries, Play feed,
      previews. WS `side_chat` action + `GET /chronicle/side-chats/...` thread
      endpoints. Memory curation deferred to the secret-scoping item so no
      unscoped side-chat atom can ever leak.)
- [x] Pin active side character in context packet.
      (Server commit `5c0cd26`. `buildSideChatPacket`: the card is the pinned
      canon sheet; retrieval scoped to what the character can know — memories
      they are subject/object of + open threads involving them — injected as
      "what they remember" / "unresolved matters" sections. Knowledge scope
      already excludes OTHER characters' private side-chat memories.)
- [x] Scope secrets by who knows them.
      (Server commit `526ec88`. Side-chat curation mints atoms with
      origin 'side_chat' + `known_by_entity_ids` = player + side character
      (+ protagonist in GM worlds, where the player speaks AS them). Side-chat
      retrieval sees only that character's own conversations; vector metadata
      carries the scope and survives edit re-embeds. Fails closed everywhere.)
- [x] Update relationship graph from side chats.
      (Server commit `f17289a`. Codex-delta extraction runs on side-chat turns
      but applies ONLY the active character's deltas; ledgered on the event
      (rewind-exact replay), meter edges re-projected, live codex frame pushed.
      Relationship memory atoms → narrative edges already flow via curation.)
- [x] Let main narration retrieve side-chat memories only when canonically
      appropriate.
      (Same commit `526ec88` — hard gate in queryRag: a private memory reaches
      main narration ONLY when the protagonist is among its knowers (GM-world
      side chats qualify; sentient-world ones don't). A secret enters the main
      story by being SHARED there, which organically mints a new main-scoped
      memory — deterministic, no model-judgment leak risk.)
- [x] Add the app surface for private side-character chats.
      (App commit `b3cea7b`. Bonds cards now expose a private-chat action; the
      thread screen loads `GET /chronicle/side-chats/:instanceId/:characterId`,
      sends WS `side_chat`, and streams `side_chat_delta` /
      `side_chat_complete` / `side_chat_error` frames. Verified by
      `flutter analyze lib`; live LLM-path verification still needs a running
      server + one real side chat.)

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

- [x] Add chapter summaries over scene summaries.
      (Server commit `b23cf21`. Every 8 scene summaries roll up into one
      LLM-generated chapter (reuses the scene-summary worker via kind:'chapter').
      `chapter_summaries` collection + index; rebuildable projection keyed by
      event range with scene provenance; full lifecycle (trigger / edit-stale +
      rebuild / rewind / reset). Children fetched by range so scene rebuilds are
      safe. CONSUMED by the prompt as of `2c4da86` (RELEVANT PAST CHAPTERS).)
- [x] Add arc summaries over plot/relationship threads.
      (Server commit `5fd9375`. Every 4 chapters roll up into one arc framed
      around plot/relationship through-lines. `arc_summaries` collection +
      index; same rebuildable-projection model + full lifecycle (trigger /
      edit-stale + rebuild / rewind / reset) as chapters; embedded into `sum_`
      and surfaced through querySummaries + the RELEVANT PAST CHAPTERS section.)
- [x] Embed summaries for semantic summary retrieval.
      (Server commit `f33694a`. Scene + chapter summaries embed into a
      per-instance `sum_<id>` Pinecone namespace on create (deterministic
      per-range vector id → rebuild overwrites). `querySummaries()` = vector
      search + Mongo status cross-check so stale summaries never surface.
      Vector lifecycle on rewind/reset/delete. CONSUMED by the prompt as of
      `2c4da86`: the context packet retrieves relevant distant summaries
      (concurrently with RAG) into a token-budgeted RELEVANT PAST CHAPTERS
      section.)
- [x] Add continuity audits comparing codex, memories, summaries, and graph
      state.
      (Server commit `541a458`. Read-only `continuityAuditService.audit()` with 8
      cross-projection checks — event-sequence integrity, single-protagonist,
      codex↔entity 1:1 linkage, memory/edge entity-ref resolution, summary bounds
      + lingering staleness (scene/chapter/arc), time + location cursor sanity —
      returning an ok/warn/fail report. `GET /admin/instances/:instanceId/
      continuity-audit`. Service is reusable by a background drift job later.)
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
- [x] Character-specific memory view.
      (Tap a character in the Bonds ledger — server commit `db59a35`, app
      commit `a9cf236`. Opens the memories that character is part of, found via
      Phase 3 entity subject/object links, `GET /chronicle/relationships/
      :instanceId/:characterId/memories`.)
- [x] "What this character remembers about you."
      (Same surface, framed as what they carry about the player — memories
      where the character is a subject/object, with emotional tone /
      relationship-shift / unresolved-thread markers.)
- [x] Promise/quest tracker.
      (Threads tab in the Lore Tome — server commit `21ec317`, app commit
      `19e52b4`. Open threads (`unresolved_thread` atoms, by importance) +
      recently-resolved ones (by close time), via `GET /chronicle/threads/
      :instanceId`. Same data that feeds the open-threads prompt section.)
- [x] Timeline branch viewer.
      (Almanac "Realities" switcher — app commit `5079600`. Lists timeline
      branches, highlights the active one, and switches the active reality via
      `PUT /chronicle/calendar/:instanceId/timeline/active`.)
- [x] Memory-aware recaps.
      (Recap tab — now the Lore Tome landing view — server commit `e43c08e`,
      app commit `8da8cb9`. "Story so far" assembled deterministically from the
      latest non-stale scene summary (prose spine) + open threads + relationship
      standings + current place/time, via `GET /chronicle/recap/:instanceId`.
      No new LLM call; always reflects current projections.)
- [x] Advanced Chronicle search and filters.
      (Echoes search + filter chips — server commit `b640134`, app commit
      `f9943d5`. `GET /chronicle/memories` gained `q` (full-text over
      text/subjects/objects, relevance-ranked), `type`, `min_importance`, and
      `unresolved` params; the Echoes tab has a search bar + Unresolved/
      Important/type chips with a no-matches state.)
