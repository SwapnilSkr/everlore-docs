# Everlore build — handoff

_As of June 10 2026, post Phase 7 (server) + audit follow-up._

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
3. **Backend first, then frontend.** Verify each: server
   `cd everlore-server && npx tsc --noEmit` (expect exit 0); app
   `cd everlore && flutter analyze lib/features/<area>` (expect "No issues found!").
4. **Commit per logical unit, directly to `main`.** Conventional commits:
   `feat(app|server)` / `fix(...)` / `perf(...)` / `docs(memory): ...`.
   **NO AI/Claude attribution lines in commit messages.**
5. **Report**: what shipped, how verified, honest gaps, recommended next step.

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
- **Phase 7 (Side-Character Chats) — SERVER COMPLETE, audited.** Commits:
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
  - **Two known gaps (honest, not bugs):** (a) the LLM-dependent paths (extraction
    NOTE framing, delta name-filter) were code-reviewed but never run against a live
    generated turn; (b) **no app surface yet** — the server contract is ready, nothing
    in Flutter triggers it.
- **Phase 10 (Product Surfaces): COMPLETE.** Lore Tome has 7 tabs: Recap (landing),
  Timeline, Echoes (searchable + filters), Almanac (calendar + timelines), Places
  (location journal), Bonds (relationship ledger → tap → "what they remember"),
  Threads (promise/quest tracker).
- **Phase 9 (Advanced Compaction): effectively complete.** Chapter summaries, summary
  embedding, summary→prompt consumption ("RELEVANT PAST CHAPTERS"), arc summaries,
  continuity audits — all done. Only **cold archival** remains, deliberately deferred
  ("only when storage pressure is real").
- Phases 1–5, 6A, 8: done earlier. See `CHECKLIST.md` for a few deferred one-offs
  (Phase 1 revision counters, Phase 2 memory→memory version links, Phase 4 broader
  BM25).

## Next (recommended order)
1. **Phase 7 app surface (RECOMMENDED).** App needs: an entry point (Bonds tab /
   character card), a thread screen consuming `side_chat_delta` / `side_chat_complete`
   / `side_chat_error` frames + the thread REST endpoints, repo + cubit per the
   Chronicle-surface pattern. WS action `side_chat` payload = `character_id` + message.
   Building this also lets you close gap (a) with one real side chat. Confirm the UI
   entry point with the user first.
2. **Phase 6B — travel** (travel events, travel-time calendar effects, permanent
   location state/facts).
3. Small: turn `continuityAuditService` into a scheduled background drift-detection
   job (the service is already reusable).
4. Deferred one-offs + Phase 9 cold archival — only when actually needed.

_Micro-opt noted, not done: 2 embeds/turn (queryRag + querySummaries run in parallel)
— could have queryRag expose its embedding to embed once._
