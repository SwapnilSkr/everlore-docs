# Everlore Server — Worker Architecture

Background jobs keep the API responsive. Uses **BullMQ** on Redis. Entry: `everlore-server/worker/index.ts`. Queues: `everlore-server/src/queues/index.ts`.

See also: [SERVICES.md](./SERVICES.md), [DATA_MODEL.md](./DATA_MODEL.md), [BILLING.md](./BILLING.md), [CONFIGURATION.md](./CONFIGURATION.md), [DEPLOYMENT.md](./DEPLOYMENT.md), [WORLD_MODEL.md](./WORLD_MODEL.md), [KINSHIP_GRAPH.md](../memory/KINSHIP_GRAPH.md).

---

## Process layout

```text
API server (src/index.ts)                 Worker (worker/index.ts)
     │ enqueue                                  │ consume
     ▼                                          ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ generation   │  │ memory-      │  │ scene-       │  │ maintenance  │
│ queue        │  │ curation     │  │ summary      │  │ queue        │
│ conc: env    │  │ conc: 5      │  │ conc: 2      │  │ conc: 1      │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

Worker startup: `connectMongo` → `connectRedis` → **warm NSFW lexicon** (`loadNsfwLexicon`) → start four BullMQ workers → register maintenance crons.

---

## Queues

| Queue | Processor | Concurrency | Rate limit | Retention (`QUEUE_RETENTION`) |
|-------|-----------|-------------|------------|-------------------------------|
| `generation` | `generation.processor.ts` (routes modes) | `env.GENERATION_CONCURRENCY` (default **3**) | `GENERATION_RATE_MAX` / min (default **10**/min) | complete ~30m / fail ~1d |
| `memory-curation` | `memory.processor.ts` | **5** | 20/min | complete ~15m / fail ~1d |
| `scene-summary` | `summary.processor.ts` | **2** | — | complete ~1d / fail ~2d |
| `maintenance` | `maintenance.processor.ts` | **1** | — | complete ~1d / fail ~7d |

Configure concurrency/rate via env — see [CONFIGURATION.md](./CONFIGURATION.md). Deploy notes: [DEPLOYMENT.md](./DEPLOYMENT.md).

### Generation job modes

Same queue and worker wrapper; `generation.processor` branches on `job.data.mode`:

| Mode | Enqueued by | Job name / data | Attempts | Processor |
|------|-------------|-----------------|----------|-----------|
| **Default turn** | `generationService.dispatch` | `generate` (no `mode`) | **2** (exponential backoff 3s) | Main path in `generation.processor.ts` |
| **`side_chat`** | `dispatchSideChat` | `side-chat`, `mode: 'side_chat'` | **1** | `side-chat.processor.ts` |
| **`replay`** | `dispatchReplay` | `replay`, `mode: 'replay'` | **1** | `replay.processor.ts` |

Job payloads may include `billingReservationId` from the WS reserve step.

---

## Lock heartbeat

Per `(playerId, instanceId)` Redis lock (`lock:gen:{playerId}:{instanceId}`) prevents overlapping turns (chat, continue, side_chat, replay).

| Constant | Value | Role |
|----------|-------|------|
| `GENERATION_LOCK_TTL_SECONDS` | **240** | Set at WS/API dispatch — covers queue wait + turn |
| `GENERATION_LOCK_HEARTBEAT_TTL_SECONDS` | **90** | Live TTL while worker processes |
| Heartbeat interval | **30s** | Refresh while turn runs |

On generation job pickup, `worker/index.ts` wraps the processor with `startGenerationLockHeartbeat`: immediately shrinks TTL to the heartbeat window, refreshes every 30s (only if lock value still matches job id), and stops in `finally`. If the worker process dies, the lock expires within ~90s instead of hanging for the full dispatch TTL. Final failure paths also `DEL` the lock after publishing terminal WS events.

---

## Billing settle / release

Ink is reserved on the API/WS before enqueue. The worker cluster owns settlement:

| Outcome | Action |
|---------|--------|
| Job **`completed`** | `billingService.settle(playerId, reservationId)` |
| Intermediate fail, **no** visible stream | Publish `generation_reset` + `generation_retrying`; keep reservation for retry |
| Intermediate fail **after** visible stream | Suppress retry (`job.discard` already set); **settle** (player saw prose); release lock |
| **Final** fail, no usable prose | Persist `dead_letter_jobs`; **`release`** reservation; publish `generation_reset` + `generation_failed`; clear lock |
| **Final** fail **after** visible stream | DLQ for diagnostics; **settle** (do not wipe player-facing scene); clear lock; **no** failure bubble that overwrites prose |

Settlement/release errors are logged only — they must not turn an already-visible story result into a worker crash. See [BILLING.md](./BILLING.md).

---

## WebSocket events (generation lifecycle)

Published on Redis `user:{playerId}:events` (relayed to the socket):

| Event | When |
|-------|------|
| Stream deltas / turn-complete frames | Normal success path (processor) |
| `generation_reset` | Before a retry replacement, or on final pre-stream failure (provisional draft) |
| `generation_retrying` | Intermediate attempt failed; more attempts remain (`attempt`, `maxAttempts`) |
| `generation_failed` | Final attempt failed with no playable prose |

Visible-stream contract: first visible prose calls `job.discard()` and sets `job.data.visibleStreamStarted = true` so BullMQ will not replace an already-shown scene with a different generation.

---

## Scheduled maintenance (cron)

Registered at worker startup on the `maintenance` queue:

| Job name | Cron | Task payload |
|----------|------|--------------|
| `decay` | `0 3 * * *` | `{ task: 'importance_decay' }` |
| `dedup-scheduler` | `0 4 * * 0` | `{ task: 'schedule_dedups' }` |
| `summary-repair` | `*/15 * * * *` | `{ task: 'repair_scene_summaries' }` |
| `continuity-audit-scheduler` | `30 2 * * *` | `{ task: 'schedule_continuity_audits' }` |
| `projection-checkpoint-scheduler` | `15 * * * *` | `{ task: 'schedule_projection_checkpoints' }` |

### Maintenance tasks (on-demand + cron fan-out)

| Task | Purpose |
|------|---------|
| `importance_decay` | Delete Pinecone vectors + archive low-importance stale memories |
| `schedule_dedups` / `dedup_memories` | Fan-out / merge near-duplicate memories (cosine ≥ 0.95; uses stored vectors) |
| `repair_scene_summaries` | Fix stuck `summary_pending` scenes |
| `repair_entity_graph` | Prune dead edge provenance, backfill card↔entity links |
| `repair_memory_links` | Repair memory↔entity / version-graph hygiene |
| `schedule_continuity_audits` / `drift_audit` | Fan-out + run read-only continuity audit; persist `meta.last_continuity_audit` |
| `schedule_projection_checkpoints` / `create_projection_checkpoint` | Fan-out + create chunked world projection checkpoints |

---

## Generation processor (default turn)

**File:** `worker/processors/generation.processor.ts`

High-level flow:

```text
1. Route mode → replay / side_chat processors when set
2. buildContextPacket()     ← RAG + summaries before codex selection
3. buildPrompt()            ← token-budgeted sections
4. NSFW routing (lexicon + mode + intent momentum)
5. callLLMStream()          ← Redis pub/sub; discard on first visible token
6. Prose hygiene / stream filters / choice-tail parse
7. extractSceneMetadata()   ← presence, location, time, choices
8. Deterministic signals    ← movement, time-skip, party, kinship, presence gaps
9. Persist event (+ travel / location / time ledgers, codex_deltas, …)
10. Fold presence; update instance cursors / party
11. Fire-and-forget: graph sync, kinship, candidates, anomalies, signal_ledger, …
12. Enqueue memory-curation (delay ~1s)
13. Enqueue scene-summary when 12-turn block complete
14. Periodic projection checkpoint nudge (e.g. every 500 sequences)
```

**Optimizations:** context in worker; location/codex/supersession don’t block TTFT; Redis session refresh; non-overlapping 12-turn scene summary blocks.

### Side chat branch

**File:** `worker/processors/side-chat.processor.ts`

- Reachability framing (`side-chat-reachability`) + `buildSideChatPacket`
- Streams in-character reply; event `type: 'side_chat'`
- Does **not** advance main time/location/scene cursors
- Codex deltas projected through `side-chat-privacy`; memory curation gets secret scope

### Replay branch

**File:** `worker/processors/replay.processor.ts`

- Streams via `memoryService.replayEvent`
- Regenerates choices + `present_characters` (+ trackable mentions) for the new variant
- WS: `replay_delta` / `replay_complete`; lock released in `finally`

---

## Memory processor

**File:** `worker/processors/memory.processor.ts`

```text
1. LLM extract 0–3 rich memory atoms
2. Resolve subject/object entity IDs
3. Side-chat: origin + known_by_entity_ids
4. Embed + Pinecone upsert (mem_{instanceId})
5. Mongo insert with provenance / rich fields
6. Relationship atoms → narrative edges as needed
7. Resolve matching open threads
8. Redis publish memories_curated
```

---

## Summary processor (3 tiers)

**File:** `worker/processors/summary.processor.ts`

| Kind | Trigger | Input |
|------|---------|-------|
| `scene` (default) | ~12 turns same `scene_tag` | Raw events |
| `chapter` | ~8 scene summaries | Scene summary texts |
| `arc` | ~4 chapter summaries | Chapter summary texts |

- Deterministic Pinecone id: `{tier}_{startSeq}_{endSeq}` in `sum_{instanceId}`
- Fetch children by event range (safe rebuilds)
- Mark stale + rebuild on event edit via covering-range helpers

---

## `worker/lib/` module map

| File | Role |
|------|------|
| `structured-output.ts` | Generation JSON / `ChoiceOption` types + sanitizers |
| `choice-tail.ts` | Parse/filter narrator `==CHOICES==` tail off the prose stream |
| `prose-stream-filter.ts` | Strip accidental JSON envelopes from the player-facing stream |
| `metadata-extractor.ts` | Post-prose LLM witness: presence, place, time, choices (split calls) |
| `choice-grounding.ts` | Deterministic drop of ungrounded tap-to-play chips |
| `choice-grounding-audit.ts` | Audit + repair choices instead of only dropping |
| `character-codex-extractor.ts` | LLM → per-turn `codex_deltas` |
| `codex-compactor.ts` | Async shrink of oversized immutable fact lists |
| `entity-adjudicator.ts` | Semantic judge for strong person-candidate terms only |
| `presence-gap-detector.ts` | Prose people missing from presence/cards → trackable mentions / stubs |
| `movement-signal.ts` | Deterministic relocation / destination / containment backstops |
| `scene-location-signal.ts` | High-confidence **initial** scene location (not a move) |
| `time-skip-signal.ts` | Deterministic narrated time-advance backstop |
| `party-signal.ts` | Travelling-with join/part detectors |
| `kinship-pattern-extractor.ts` | Deterministic kinship assertions from player + prose |
| `kinship-transition-extractor.ts` | Lifecycle transitions (deceased, estranged, …) |
| `kinship-hygiene.ts` | Inverse-close, confidence, competing-assertion cleanup |
| `kinship-epithet-resolver.ts` | Residue epithet → existing role card (micro-LLM) |
| `premise-kinship-extractor.ts` | One-time premise / system_seed kinship extraction |
| `relation-candidate-detector.ts` | Narrator-only review candidates (not canon) |
| `canon-revision-detector.ts` | Conservative identity/kinship revision proposals |
| `projection-anomaly-detector.ts` | Prose vs projection inconsistency findings |
| `signal-ledger.ts` | Build FP/FN tallies for `signal_ledger` rows |
| `nsfw-classifier.ts` | Lexicon load/cache + scene scoring / borderline intent |
| `side-chat-privacy.ts` | Fail-closed projection of side-chat codex deltas |
| `side-chat-reachability.ts` | Dead / present / nearby / remote / seek framing |
| `template-cast-extractor.ts` | One-time authored cast extraction for templates |

---

## Dead letter & failure

Failed generation jobs on the **final** attempt (with no successful visible-stream settlement path that skips UX failure) insert into `dead_letter_jobs` and publish `generation_failed` when appropriate. Lock is always cleared on terminal failure. Scene-summary failures are logged with instance/scene/range context.

---

## Verification

| Command / endpoint | Purpose |
|--------------------|---------|
| `bun run scripts/rewind-audit.ts` | Integration: clone instance, rewind, projection assertions |
| `bun run audit:choices` | Live LLM audit: chips + presence |
| `bun run audit:location` | Live audit: travel / places / presence fold |
| `bun run audit:codex-dedup` | Live audit: kin-epithet → existing card |
| `bun run audit:replay-edit` | Replay + edit + variant select |
| `GET /admin/instances/:id/continuity-audit` | On-demand drift audit |
