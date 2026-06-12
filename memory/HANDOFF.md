# Everlore build — handoff

_As of June 12 2026, post **Phase 2 memory version-links (Slice 1)** — see latest
State bullet — and (June 11) Phase 6B (travel + location state/facts + travel UI marker),
Phase 7 (server + app surface + audit follow-ups), scheduled continuity drift
detection, and a June-11 extractor-drift / dedup hardening pass (duplicate-character
fix `9101349` + scale-robust location resolution `a12e94e`), plus Location Graph
P0–P2 (nested atlas) and a **P2.6 movement/presence accuracy hardening** pass
(deterministic backstops for the "I go to my room but I'm still in the dining room"
class). Phases 1–10 are complete; Phase 2 memory-version links and Phase 4 broader
BM25 are intentionally reopened for the next planning pass._

This is the durable, in-repo handoff for whoever (human or AI agent) picks up the
"infinite memory" build next. Read this, then `CHECKLIST.md` (the authoritative
task list) and `MEMORY_ARCHITECTURE.md` / `PROJECTION_AND_MUTATION_MODEL.md`.

> **🐞 Open bug backlog (2026-06-12 QA run):** `PLAYTEST_FINDINGS_MERGED_2026-06-12.md`
> — ~48 live findings from the 6-agent parallel playtest, deduped into 13 root-cause
> clusters with the suspected code seam + fix per cluster. **P0 = side-chat secret
> leak into main projections + identity-boundary collapse (player facts/cards in
> sentient/character worlds).** Structural audits stayed 8/8 green throughout — the
> corruption is semantic and live-only. Run the loop via `AUTOCHAT_PLAYBOOK.md`.

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
- **Live-turn verification (Tier 2):** see `LIVE_VERIFICATION.md` — the runbook for
  stress-testing LLM-dependent behaviour against REAL generated turns (worker +
  `generationService.dispatch`, inspect Mongo, `rewindToSequence` to restore). READ IT
  before claiming any witness-seam feature works: `typecheck` + `audit:*` green is
  necessary but NOT sufficient (the June pass found 5 bugs the audits missed). It
  documents the false-positive traps (stale `session:<iid>` cache, stale worker code,
  async-projection lag, extractor-sees-only-AI-prose, LLM stochasticity) and the
  stress matrix + cleanup discipline.

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
- **Location Graph — closed out (June 12 2026).** Everything buildable + verifiable
  without new content is DONE; the rest is content-gated with concrete reopen triggers.
  - **Subtree `world_root_id` refresh on cross-root re-parent — DONE (server `c3b04f0`).**
    The shipped P1 KNOWN LIMIT: the cartographer set `parent_id` on a re-parent but never
    refreshed the denormalized top-of-chain `world_root_id` down the subtree, so a future
    cross-root re-parent would strand descendants on a stale realm and bleed place-recall
    (no-op in single-world → LATENT; fixed before it can corrupt — same call as the
    intra-world collision fix). `entityGraphService.refreshSubtreeWorldRoot`: deterministic,
    idempotent BFS down `parent_id`; a self-rooted DESCENDANT (nested world) is never
    rerooted + stops the descent; only the start node sheds its root; bounded depth guards
    cycles; null target = implicit single world. The re-parent reveal in `placeLocation`
    now calls it (parent's `world_root_id`, or null — never the parent's bare id), making
    "re-parent keeps `world_root_id` correct" an enforced invariant. Verified:
    `audit:location-resolution` +8 cases + full suite + rewind-audit + location-audit green.
  - **P2.5 relation edges, P3 multi-world, open-world limits #2–4 — DEFERRED, content-gated.**
    Each has a concrete reopen trigger in `LOCATION_GRAPH.md` (P2.5/P3 sections +
    "Open-world limits"). NOT built ahead of content (same discipline as Phase 4 BM25):
    the relation-edge producer would show nothing + be unverifiable today; the multi-world
    spine is ready to be *exercised* when a second realm appears, not built speculatively.
- **Phase 2 memory version-links (Slice 1) — DONE (server `f6a1e81`, June 12 2026).**
  Captures the write-time supersession lineage the codex-retirement path was
  discarding (chosen FIRST over Phase 4 BM25: the BM25 gap is scale-gated +
  unmeasurable at ~26 turns, while this signal is lost on every deferred turn).
  `memory.model` gains `updates_/extends_/derives_from_memory_ids` (only `updates`
  has a producer; the other two reserved/inert) + a `superseded_by_event_ids`
  backward mark. The supersession service stamps that race-free single-writer mark
  on archived atoms, returns the superseded ids, and **scopes Pinecone matches to
  MAIN origin** so a main codex retirement never evicts/links a private side-chat
  secret (a privacy improvement on top of the feature). The curator
  (`memory.processor`) materializes the forward `updates_memory_ids` on the turn's
  new correcting atoms from those marks. Lifecycle (links are a projection):
  `pruneMemoryVersionLinks` `$pull`s removed ids from surviving atoms on rewind +
  edit-recuration; rewind also drops backward marks for removed events;
  `repair_memory_links` maintenance task reconciles the supersession/curation race
  (materialize forward links from backward marks) + prunes dangling links/marks,
  idempotent. `GET /admin/events/:eventId/projections` surfaces the lineage per
  atom, **`allowsKnowledge`-gated (fail closed)**. Verified: `audit:memory-links`
  (Tier 1, throwaway instance — reconcile + both prunes + idempotency) 9/9; tsc
  clean; rewind-audit green. **TIER-2 LIVE-VERIFIED (June 12 2026):** drove real
  turns through the worker — (a) a natural fact-reversal fired the codex `retire_state`
  → supersession invoked WITH the event id (threading works), but its ≥0.82 match found
  nothing for the short retired-state phrasings → archived 0 → correctly no marks/links;
  (b) the full archive→mark→materialize chain run on LIVE infra via a throwaway atom
  (real embed → Pinecone ≥0.82 match → `superseded_by_event_ids` stamped →
  `repair_memory_links` reconcile materialized the forward `updates_memory_ids`); (c) the
  live rewind pruned a dangling link (`pruneMemoryVersionLinks` confirmed live) and
  restored EXACT baseline. HONEST RESIDUAL: end-to-end linking on a NATURAL turn is
  probabilistic (gated on supersession's pre-existing 0.82 match); the link code itself is
  fully exercised. **Phase 4 BM25 deferred-with-a-gate:** the
  recall-gap measurement, decision rule, and scale gate are captured in
  `RETRIEVAL_MEASUREMENT.md` (do NOT run the eval on the 26-turn instance — false
  negative). Slice 2 `extends` + Slice 3 `derives_from` remain deferred (no producer).
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
  - **LIVE-VERIFIED (June 11 2026):** the new extractor fields were finally driven
    against real generated turns and exposed two gaps, both fixed: `time_elapsed` is
    lost when the player narrates a skip the AI prose doesn't restate (extractor sees
    only AI prose) → deterministic `time-skip-signal.ts` backstop over player input
    (calendar D1→D8 on "Weeks pass"); `location_state_changes` under-detected positive
    transformations → broadened prompt (garden → "roses have revived and bloom").
    travel-via-location already worked (cursor + travel markers fired correctly).
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
- Phases 1–5, 6A, 8: done earlier. Phase 2 memory→memory version links and Phase 4
  broader BM25 are now reopened for discussion/planning. Phase 1 revision counters
  and Phase 9 cold archival remain closed as deferred-by-design.
- **Location Graph P2.6 — resolution accuracy / movement hardening — DONE
  (June 11 2026).** Fixes the user's "I go to my room but I'm still in the dining
  room (with my parents)". Root cause: the small-model witness under-reports BOTH
  `viewpoint_moved` AND the name of a personal space, so the cursor stuck on the
  place left and presence carry-forward never reset — location + `present_characters`
  are ONE detection failure coupled through the F3 `sceneBroke` reset, not two bugs.
  Fuzzy matching was a red herring (it never got a fair input); the fix is at the
  witness seam with deterministic server math (same pattern as F3 presence + codex
  name-grounding). `worker/lib/movement-signal.ts` (`detectNarratedMovement` +
  `resolvePossessiveRoomName`, pure + `audit:movement` 25/25) feeds
  `generation.processor`: owner-scoped name override on a possessive retreat
  ("my room" → "<owner>'s room", placed lateral), `viewpoint_moved` corroboration
  gated on a real place change, and presence reset on any resolved-ENTITY change.
  Live dev instance repaired with `repair-bedroom-anchor.ts` (minted "Swapnil
  Sarkar's room" under the mansion, re-anchored seq 25-26, cleared stale presence,
  repointed cursor). typecheck + `audit:location` + `audit:location-resolution`
  green. **LIVE-VERIFIED** (drove real LLM turns through the worker): "I go to my room
  and shut the door" → cursor → "Swapnil Sarkar's room", `present:[]`, correct travel
  marker, entity reused (no dup); round-trip + stay-put feint all correct. Verification
  CAUGHT + fixed three flaws unit audits missed — (a) possessive namer grabbed the
  ORIGIN room ("leave my room and head to the dining room") → departure-context guard;
  (b) `isVagueLocationLabel` missed possessive-pronoun rooms ("his room") so the
  memory-curation resolver minted ghost atlas nodes → broadened guard + applied it in
  `resolveOrCreateEntities`; (c) `repair-bedroom-anchor` didn't bust the Redis
  `session:<iid>` cache (stale cursor for the next turn). All fixed/committed; dev
  instance rewound to seq 26. STILL UNVERIFIED LIVE: Phase 7 side-chat + Phase 6B
  time-advance / location_state-facts.
- **Extractor-drift / dedup hardening pass (June 11 2026) — DONE.** All in the
  `CHECKLIST.md` Bug Fixes section with detail:
  - `9101349` — **codex extractor inventing duplicate character cards.** A secretive
    father called only "the man" in the prose was minted as a separate "Mysterious
    Man" card (a name found NOWHERE in the prose, coined from mood). Fix made
    non-negotiable: prompt rules (never invent a name; bare descriptor → present-cast
    member) + `presentCast` threaded from the generation + side-chat processors + a
    DETERMINISTIC backstop that drops any new-card delta whose name isn't grounded as
    a phrase in the turn text once a roster exists. `codex-dedup-audit.ts` gains
    scenario B (secretive "the man" → Father) + C (a new NAMED stranger still mints a
    card). Live instance repaired (merge + ledger/presence scrub → zero trace);
    `merge-character-cards.ts` hardened (per-edge collision handling + idempotent fold).
    KNOWN LIMIT: the memory-curation path can still mint an un-carded character ENTITY
    (graph-only, roster-nudged) — deferred with user's OK.
  - `a12e94e` — **scale-robust location resolution.** `resolveLocationAnchor` no
    longer loads the full entity registry per turn: indexed exact lookup + a bounded
    token-similarity pass so long-tail returns ("the garden" → "Night Garden") dedupe
    outside the 30-name roster, with a conservative threshold keeping distinct places
    apart. New `audit:location-resolution` (pure scoring + DB integration on a
    throwaway instance, all invariants held). Complements the `a27cc8f` KNOWN PLACES
    roster (the hot-set nudge).

## Next (recommended order)
Phases 1–10 are complete; the **Location Graph** initiative is closed out (everything
buildable-without-content done; P2.5/P3/open-world-limits content-gated with reopen
triggers in `LOCATION_GRAPH.md`); **both reopened planning items are resolved** —
Phase 2 version-links Slice 1 shipped (`f6a1e81`), Phase 4 BM25 deferred with a
measurement gate (`RETRIEVAL_MEASUREMENT.md`). What's genuinely OPEN now:
- **Tier-2 live checks — DONE (June 12 2026 live pass).** Drove real worker turns
  against the dev instance, restored to exact baseline (rewind to seq 27 + cache bust):
  version-link supersession trigger + full archive→mark→materialize chain + live
  link-prune on rewind; Phase 7 side-chat (shared-ledger event, frozen story date +
  cursor, `origin:'side_chat'` memory with `known_by=[player,character,protagonist]`,
  streamed reply); Phase 6B time-advance ("Weeks pass" → calendar D1→D8) — bonus:
  re-confirmed P2.6 movement (room→hallway travel, cursor moved, presence reset, entity
  reused). Remaining live gap = **Tier 3 only** (in-app Flutter UI — the user's pass).
  `location_state` positive-transformation remains probabilistic (not re-tested here).
- **Content-gated, pull in on demand:** Location Graph P2.5 relation edges, P3
  multi-world, open-world limits #2–4 (travelling-party presence, dedup-at-scale, mobile
  containers, parallel scenes); Phase 2 version-link Slices 2/3 (`extends`/`derives_from`,
  no producer); Phase 4 BM25 (run the gate eval only at scale).
- **Still deferred-by-design:** Phase 1 revision counters, Phase 9 cold archival.

Historical order (kept for context):
0. **Location Graph P0 + P1 — DONE** (`4483949`, `f95099d`). P0: vague-label guard
   (generic/relative labels never mint). P1: the containment spine
   (`parent_id`/`world_root_id`/`place_kind` + unique index incl. world_root_id so
   same-named places coexist across realms), witness fields (`containment_hint`/
   `movement`), the server cartographer (`placeLocation`: deeper/out/lateral/world_shift
   + re-parent reveal + world-root minting), world-root-scoped resolution, audit, and a
   dev-instance backfill (8 locations under a mansion building + exterior). **P2 — atlas
   UI — DONE** (`e633956` + app `c8d7bbc`): `listLocations` returns the full spine, the
   Places tab is a fog-of-war nested tree (place-kind glyphs, current place highlighted +
   auto-expanded, fold/unfold, tap → journal); also hardened `merge:location` to re-point
   memory place-anchors (+ `repair-orphan-place-anchors.ts`). **NEXT for this initiative:
   P2.5/P3** — non-containment relation edges (`mirror_of`/`allied_with`/`borders`; needs
   an extractor pass to MINT them, then surface), multi-world maturity, and the deferred
   subtree `world_root_id` refresh on cross-root re-parent. Full design + locked decisions
   in `LOCATION_GRAPH.md`. **P2.6 — resolution accuracy / movement hardening — DONE**
   (movement-signal backstops; see State above): inserted before P2.5 because a wrong
   cursor poisons place-recall RAG, so accuracy precedes relation-edge cosmetics.
1. **Live-turn verification pass — DONE.** All three targets driven through the real
   worker. **(P2.6)** cursor/presence/travel/dedup correct (3 bugs found + fixed).
   **(Phase 7)** side-chat: in-character reply, story time + cursor FROZEN, main reads
   skip the `side_chat` event, memory scoped `origin:side_chat` + `known_by=[player,
   character,protagonist]` (GM world) — clean. **(Phase 6B)** travel/cursor work; found
   + fixed two extraction gaps — time-advance was lost when the player narrated a skip
   the AI prose didn't restate (extractor reads only AI prose), fixed with
   `time-skip-signal.ts` deterministic backstop over player input (verified: "Weeks
   pass" → calendar D1→D8); and `location_state` under-detected positive
   transformations (destructive-only prompt examples), fixed with a broadened prompt
   (verified: garden → "the roses have revived and bloom vibrantly"). Recipe that
   worked: start `bun run worker/index.ts` (streams via `redis.publish`, NO HTTP/WS
   needed), enqueue via `generationService.dispatch(...)` / `dispatchSideChat(...)`,
   poll Mongo, then `rewindToSequence` to restore + `del session:<iid>` after any
   out-of-band instance write. **Everything Phases 1–10 + Location Graph P0–P2.6 is now
   live-verified.**
2. **Phase 2 memory-version links — Slice 1 DONE** (`f6a1e81`; supersession-produced
   `updates_memory_ids` + prune lifecycle + `repair_memory_links` reconcile + admin
   lineage). Remaining for this item: a NATURAL-fact-reversal Tier-2 live check
   (preconditions absent in the dev instance — see State), and Slices 2/3
   (`extends_memory_ids` needs a refine-not-replace judgment; `derives_from_memory_ids`
   needs an inference producer that doesn't exist yet).
3. **Phase 4 broader BM25 — DEFERRED with a gate.** Decided not to build ahead of a
   demonstrable recall gap, which is scale-gated and unmeasurable on the 26-turn dev
   instance. The measurement, decision rule (≥10pp slice lift, no precision regress),
   scale gate, and privacy scoping are in `RETRIEVAL_MEASUREMENT.md`. Pull in when a
   real instance reaches scale; first slice = eval harness + baseline, not the index.
4. **Open-world limits** (`LOCATION_GRAPH.md` "Open-world limits" + CHECKLIST). The
   intra-world same-name **collision is now FIXED** (area-scoped resolution + the
   unique index gained `parent_id`; `idx_entities_instance_type_root_parent_name`, old
   one dropped at startup — live-migrated, no dup-key; `resolveAreaId` walks to the
   nearest settlement/building so `tavern@Ashford` ≠ `tavern@Riverton` while
   same-building returns still resolve). Multi-settlement content is safe to ship. The
   REMAINING limits are content-driven: traveling-party presence (a real feature —
   `travelling_with` set), dedup-at-scale (30-roster cap), mobile containers / parallel
   "meanwhile" scenes. Stack degrades gracefully on all of them.
5. **Still deferred:** Phase 1 revision counters and Phase 9 cold archival. Reopen
   only for a concrete consumer or real storage pressure.

_Micro-opt noted, not done: 2 embeds/turn (queryRag + querySummaries run in parallel)
— could have queryRag expose its embedding to embed once._
