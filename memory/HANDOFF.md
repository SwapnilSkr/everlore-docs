# Everlore build — handoff

_As of June 10 2026, post Phase 6B (travel + location state/facts + travel UI marker),
Phase 7 (server + app surface + audit follow-ups), and scheduled continuity drift
detection. Phases 1–10 are complete; remaining one-offs are closed as
deferred-by-design._

This is the durable, in-repo handoff for whoever (human or AI agent) picks up the
"infinite memory" build next. Read this, then `CHECKLIST.md` (the authoritative
task list) and `MEMORY_ARCHITECTURE.md` / `PROJECTION_AND_MUTATION_MODEL.md`.

## Repos
- `everlore/` — Flutter/BLoC app.
- `everlore-server/` — Bun/Elysia server + BullMQ workers.
- `everlore-docs/` — this docs repo.
- The parent `rpg-ai/` is **not** a git repo; the three subdirs each are. Commit
  in the relevant repo(s); use `git -C <abs-path>` (cwd resets between shells).

## How to work (the rhythm)
1. **One checklist item at a time.** Confirm direction with the user only on a real
   design fork; otherwise proceed.
2. **Edit `CHECKLIST.md` BEFORE and AFTER each item.** Before: flip to
   `[~] (IN PROGRESS)` + a one-line note. After: `[x]` + commit ref(s) + 2–3 line
   summary. Commit the checklist edit as `docs(memory): ...`.
3. **Update this `HANDOFF.md` after EVERY pass / slice / item — not just at the end
   of a phase.** This is a hard requirement, called out because past agents have
   skipped it. If you are a Claude agent (or any AI): you may keep your own private
   memory files (e.g. `~/.claude/.../memory/session_handoff.md`), but **do not treat
   those as a substitute for this file.** Your private memory is invisible to the
   next agent and to the user. THIS file is the single durable, in-repo source of
   truth. So after each slice you ship: edit `CHECKLIST.md` (per rule 2) AND edit
   this `HANDOFF.md` — update the date/title line, the relevant State bullet (exact
   commit refs, what shipped, how verified, honest gaps), and the Next list — then
   commit it as `docs(memory): ...`. A reader of this file alone should be able to
   resume without reconstructing the prior session from git history or from any
   agent's private notes.
4. **Backend first, then frontend.** Verify each: server
   `cd everlore-server && npx tsc --noEmit` (expect exit 0); app
   `cd everlore && flutter analyze lib/features/<area>` (expect "No issues found!").
5. **Commit per logical unit, directly to `main`.** Conventional commits:
   `feat(app|server)` / `fix(...)` / `perf(...)` / `docs(memory): ...`.
   **NO AI/Claude attribution lines in commit messages.**
6. **Report**: what shipped, how verified, honest gaps, recommended next step.

## Reusable patterns
- **New Chronicle surface:** server read method (often on an existing service) →
  controller method → route in `chronicle.routes.ts` (ownership-checked via
  `worldInstances().findOne({_id, player_id})`); app: `data/<x>.dart` Equatable
  model (`fromJson`), repo method in `chronicle_repository.dart`, a `ChronicleTab`
  enum value + state field + `loadX()` + `switchTab` branch in
  `chronicle_cubit.dart`, a `widgets/<x>_view.dart`, wired into
  `chronicle_screen.dart`. Theme = `EverloreTheme`. The Lore Tome tab bar scrolls
  horizontally.
- **Summary tiers (Phase 9):** events → scenes (12 turns) → chapters (8 scenes) →
  arcs (4 chapters). All in `worker/processors/summary.processor.ts`, discriminated
  by `job.data.kind` on the ONE `scene-summary` queue (no new workers). Each tier is
  a rebuildable projection keyed by `event_range`; **fetch children BY RANGE** (not
  id — rebuilds mint new `_id`s), replace-on-range, deterministic Pinecone id
  `<tier>_<start>_<end>` in namespace `sum_<instanceId>`. Lifecycle parity: trigger
  after child completes, stale+rebuild via `staleSummariesCoveringEvent`, rewind
  delete (vectors+docs), reset/delete in `deletion.service`.
- **Retrieval:** `queryRag` (vector+keyword+entity+location+threads, RRF) and
  `querySummaries` in `src/providers/rag.provider.ts`. Both feed
  `context-packet.service.ts` (runs in the WORKER before codex selection) →
  `prompt-builder.ts` token-budgeted sections (`SECTION_TOKEN_BUDGET` +
  `REFERENCE_SHARE` must sum to 1.0; recents have a hard floor).
- **Verification harnesses (read-only, trusted):** `scripts/rewind-audit.ts`
  (27+ assertions) and `continuityAuditService`
  (`GET /admin/instances/:id/continuity-audit`).

## Privacy / exclusion invariant (Phase 7)
`side_chat` events share the event ledger + sequence counter, so **any read of the
events collection that feeds a MAIN-story surface must filter
`type: { $ne: 'side_chat' }`**. Known exclusion sites: `summary.processor` (scene
rollup), `generation.service` (main recents + Play feed), `context-packet` (worker
recents), `instance.service` (previews), `memory.service.getEvents` (Timeline),
`memory.service.buildRecap` (spine fallback), `location.service` (Location Journal),
and `time.service` (Calendar). When adding a main-story read, grep
`events().find`/`findOne`/`aggregate` and confirm the filter. Memory *secrets* are
gated separately by `origin:'side_chat'` / `known_by_entity_ids` in `queryRag`
(fail closed) — see `rag.provider.ts`, `context-packet.service.ts`, plus the
Recap/Echoes/Threads/Location read filters.

## State (what's done)
- **Phase 6B (Travel + location state/facts) — COMPLETE.** Extraction-driven
  (confirmed with the user), NOT an explicit travel action — it matches how
  `scene_tag` / `present_characters` / `current_location` are already derived from
  finished prose. Commits:
  - `723d366` — **travel events + narrated time advances the calendar.** A player
    turn whose resolved end-of-turn location differs from the prior cursor becomes a
    `type:'travel'` event with `data.travel={from,to}` (detected server-side by
    comparing locations — no model self-report). Travel is a MAIN-story event
    (continue/wait ticks stay `calendar_tick`). The scene-metadata extractor emits
    `time_elapsed`; on a real turn ANY narrated skip ("weeks passed") now advances
    the day-level calendar via `data.time_advanced`, not just the explicit wait tick.
    `advanceDays` parses `<amount> <unit>` (digits or worded numbers) across
    day/week/month/season/year.
  - `3e71e4b` — **location state + permanent facts.** Location entities carry
    provenance-tracked `location_state` (mutable condition, ring-capped) +
    `location_facts` (enduring canon, append-bounded), each a
    `LocationFactDoc{text,source_event_id,source_sequence,created_at}`, applied by
    `entityGraphService.applyLocationFacts` from the extractor's
    `location_state_changes` / `location_permanent_facts`. **Event-sourced and
    rewind-safe:** rewind range-prunes `source_sequence>=seq` (inline in
    `repairAfterRewind`), edit/replay prunes by `source_event_id`
    (`pruneLocationFactsByEvents`, called in `recurateMemoriesForEvent`). reset/delete
    need no change (facts live on the entity; `deleteInstanceData` drops it). Surfaced
    in the prompt's CURRENT LOCATION section + the place journal
    (`permanent_facts` / `current_state`); app journal screen renders them
    (`everlore@c8cd9a1`).
  - **Verified:** tsc clean (both server commits); `flutter analyze lib/features/chronicle`
    clean; rewind-audit 27/27 unbroken; throwaway location-facts smoke 8/8
    (apply/provenance/dedup/edit-prune/rewind-prune), then deleted.
  - `2e4daa9` + `deb2bfd` — **travel UI marker polish.** Calendar events expose
    `travel:{from,to}`; the app parses that payload and renders "Traveled from X
    to Y" in Almanac dated events and above travel turns in the main
    Timeline/NarrativeBubble. Verified with server `bun run typecheck` and app
    `flutter analyze lib`.
  - **Known gap (honest):** the new extractor fields (`time_elapsed`, travel-via-location,
    `location_*_changes`) are schema/prompt-correct + apply-path-tested but never run
    against a live generated turn.
- **Phase 7 (Side-Character Chats) — SERVER + APP SURFACE COMPLETE, audited.** Commits:
  - `e07a6b6` — `side_chat` event type; shared ledger/sequence; streamed in-character
    reply built from the codex card; **story time does not advance**. Excluded from
    main recents, scene summaries, Play feed, previews. WS action `side_chat`; REST
    `GET /chronicle/side-chats/:instanceId[/:characterId]`.
  - `5c0cd26` — `buildSideChatPacket` pins the active card + scopes recall to what
    they can know (their memories + open threads involving them).
  - `526ec88` — secret scoping: memories minted with `origin:'side_chat'` +
    `known_by_entity_ids`; `queryRag` hard gate so a private memory reaches main
    narration ONLY if the protagonist is a knower; **fails CLOSED** in both the Mongo
    clause and the vector-metadata-only fallback. GM worlds add the protagonist to
    `known_by` (player speaks AS them); sentient worlds don't.
  - `f17289a` — codex deltas from side chats apply ONLY the active card; ledgered so
    rewind replays them; meter edges re-project onto the graph.
  - `a960a6b` — **audit fix**: `buildRecap`'s latest-event fallback wasn't excluding
    side chats → could leak a private reply as the Recap spine in early playthroughs.
    Now scoped to non-`side_chat`.
  - `f4bb003` — **audit follow-up fix**: Location Journal and Calendar event reads
    also needed main-surface exclusion; Recap open-thread memories and Location
    Journal memories now use main-visible memory filtering so private side-chat
    atoms only surface when the protagonist is among `known_by_entity_ids`.
  - `bf9ff3e` — **Lore Tome consistency fix**: Timeline (`getEvents`) excludes
    `side_chat` events; Echoes (`getMemories`) and Threads (`listThreads`) use the
    same main-visible memory gate as Recap/Location, so all Lore Tome tabs are
    main-story scoped except the dedicated side-chat thread surface.
  - `b3cea7b` — **app surface**: Bonds cards expose a private-chat action; the
    side-chat thread screen loads REST history, dispatches WS `side_chat`, and
    streams `side_chat_delta` / `side_chat_complete` / `side_chat_error`.
  - **Known gap (honest, not a bug):** the LLM-dependent paths (extraction NOTE
    framing, delta name-filter, streamed reply) were code-reviewed and app-wired
    but still need one live generated side chat against a running server.
- **Background drift detection — DONE (server `c56b9c3`).** A daily
  `continuity-audit-scheduler` repeatable job (cron `30 2 * * *`, registered in
  `worker/index.ts`) fans out per-instance `drift_audit` jobs across active worlds
  with real history (`meta.total_events > 5`, not archived; capped 1000/run). Each
  runs the existing read-only `continuityAuditService.audit()`, structured-logs any
  warn/fail under `continuity.drift`, and records a compact result on the instance
  (`meta.last_continuity_audit`: healthy / summary{ok,warn,fail} / max_sequence /
  issues[] / checked_at) so drift is visible to an admin without re-running the
  audit. **Detection only — no mutation/auto-repair** (confirmed with the user).
  Verified: tsc clean; throwaway smoke drove `drift_audit` against a live instance —
  audit ran, warn logged, `meta.last_continuity_audit` persisted, then deleted.
  - `076a365` — **admin status surface**:
    `GET /admin/instances/continuity-audits` lists compact instance audit status
    with `status=all|healthy|unhealthy|missing|stale`, paging, and `stale_days`.
    This surfaces scheduled results without running a fresh audit. Verified by
    server `bun run typecheck`.
- **Phase 10 (Product Surfaces): COMPLETE.** Lore Tome has 7 tabs: Recap (landing),
  Timeline, Echoes (searchable + filters), Almanac (calendar + timelines), Places
  (location journal), Bonds (relationship ledger → tap → "what they remember"),
  Threads (promise/quest tracker).
- **Phase 9 (Advanced Compaction): effectively complete.** Chapter summaries, summary
  embedding, summary→prompt consumption ("RELEVANT PAST CHAPTERS"), arc summaries,
  continuity audits — all done. **Cold archival** is closed as deferred-by-design
  until real Mongo storage pressure or cost data justifies it.
- Phases 1–5, 6A, 8: done earlier. Former open one-offs are now closed as
  deferred-by-design: Phase 1 revision counters, Phase 2 memory→memory version
  links, Phase 4 broader BM25, and Phase 9 cold archival. Reopen only when there is
  a concrete consumer, measured search/retrieval gap, or real storage pressure.

## Next (recommended order)
Phases 1–10 are complete and `CHECKLIST.md` has no unchecked rows. What remains is
runtime verification and reopen-only maintenance:
1. **Live-turn verification pass (RECOMMENDED).** Several LLM-dependent paths are
   code-correct but never run against a real generated turn. Start server + worker +
   app and confirm: **(Phase 7)** Bonds → private chat streams a reply, REST reload
   shows it, and side-chat curation doesn't leak into the main Lore Tome tabs;
   **(Phase 6B)** a turn that narrates travel/time produces a `travel` event, moves
   the calendar date, and records `location_state` / `location_facts` on the place
   (visible in the journal).
2. **Reopen-only deferred work** — Phase 1 revision counters, Phase 2 memory→memory
   version links, Phase 4 broader BM25 index, Phase 9 cold archival. Build only
   when there is a concrete consumer, measured retrieval/search gap, or real
   storage pressure.

_Micro-opt noted, not done: 2 embeds/turn (queryRag + querySummaries run in parallel)
— could have queryRag expose its embedding to embed once._
