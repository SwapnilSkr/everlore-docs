# Everlore Server — Worker Architecture

Background jobs keep the API responsive. Uses **BullMQ** on Redis.

See also: [SERVICES.md](./SERVICES.md), [../system-guide/02-one-turn-journey.md](../system-guide/02-one-turn-journey.md), [../code-reference/SERVER.md](../code-reference/SERVER.md).

---

## Process layout

```text
API server (src/index.ts)          Worker (worker/index.ts)
     │ enqueue                           │ consume
     ▼                                   ▼
┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌─────────────┐
│ generation  │  │ memory-      │  │ scene-      │  │ maintenance │
│ queue       │  │ curation     │  │ summary     │  │ queue       │
│ conc: 3     │  │ conc: 5      │  │ conc: 2     │  │ conc: 1     │
└─────────────┘  └──────────────┘  └─────────────┘  └─────────────┘
```

Worker startup also **warms NSFW lexicon** from Mongo (`loadNsfwLexicon`).

---

## Queues

| Queue | Processor | Concurrency | Rate limit |
|-------|-----------|-------------|------------|
| `generation` | `generation.processor.ts` | 3 | 10/min |
| `memory-curation` | `memory.processor.ts` | 5 | 20/min |
| `scene-summary` | `summary.processor.ts` | 2 | — |
| `maintenance` | `maintenance.processor.ts` | 1 | — |

**Generation queue job modes:** default turn, `replay`, `side_chat` (same worker, different processor branch).

---

## Scheduled maintenance (cron)

| Job | Cron | Task |
|-----|------|------|
| `decay` | `0 3 * * *` | Archive stale low-importance memories |
| `dedup-scheduler` | `0 4 * * 0` | Fan out per-instance dedup |
| `summary-repair` | `*/15 * * * *` | Re-queue stuck summaries |
| `continuity-audit-scheduler` | `30 2 * * *` | Fan out drift audits (max 1000 instances) |

### On-demand maintenance tasks

| Task | Purpose |
|------|---------|
| `importance_decay` | Delete Pinecone vectors + archive Mongo docs |
| `schedule_dedups` / `dedup_memories` | Merge near-duplicate memories (cosine ≥ 0.95); **fetches stored vectors** instead of re-embedding |
| `repair_scene_summaries` | Fix stuck `summary_pending` scenes |
| `repair_entity_graph` | Prune dead edge provenance, backfill card↔entity links |
| `schedule_continuity_audits` / `drift_audit` | Run read-only consistency audit; log + persist `meta.last_continuity_audit` |

---

## Generation processor (main turn)

**File:** `worker/processors/generation.processor.ts`

### Flow (current — June 2026)

```text
1. buildContextPacket()     ← RAG + summaries BEFORE codex selection
2. buildPrompt()            ← token-budgeted sections
3. callLLMStream()          ← stream to client via Redis pub/sub
4. repairProseHygiene()
5. extractSceneMetadata()   ← protagonist + KNOWN CAST roster + KNOWN PLACES; choices anchored to protagonist POV; viewpoint_moved, characters_departed, present_characters
6. Persist event (+ travel only when viewpoint_moved + different place; location cursor follows current_location whenever set)
7. Fold presence: continuous scene = prior ∪ thisTurn − departed; scene break = fresh list
7. Fire-and-forget: location facts, codex extract, graph sync, supersession
8. Enqueue memory-curation (delay 1s)
9. Enqueue scene-summary if 12-turn block complete
```

**Optimizations:**
- Context built in worker (not API) so retrieval precedes codex pins
- Location facts / codex compaction / supersession **don't block** streaming
- Redis session refresh (1h TTL)
- Scene summaries: **non-overlapping** 12-turn blocks

### Side chat branch

**File:** `worker/processors/side-chat.processor.ts`

- `buildSideChatPacket` — scoped memories + threads
- Streams in-character reply
- Event type `side_chat` — **does not** advance main time/location/scene
- Still runs codex deltas for active character + secret-scoped memory curation

### Replay branch

**File:** `worker/processors/replay.processor.ts`

- Streams via `memoryService.replayEvent`
- Regenerates **choices + present_characters** for the new variant (same extractor as primary turns)
- `replay_complete` WS includes top-level chips/presence and per-variant `choices` / `present_characters`
- Lock released in `finally`

---

## Memory processor

**File:** `worker/processors/memory.processor.ts`

```text
1. LLM extract 0–3 rich memory atoms
2. Resolve entities (subject/object IDs)
3. Side-chat: set origin + known_by_entity_ids
4. Embed + Pinecone upsert (mem_{instanceId})
5. Mongo insert with provenance
6. Relationship atoms → narrative edges
7. Resolve matching open threads
8. Redis publish memories_curated
```

---

## Summary processor (3 tiers)

**File:** `worker/processors/summary.processor.ts`

| Kind | Trigger | Input |
|------|---------|-------|
| `scene` (default) | 12 turns same `scene_tag` | Raw events |
| `chapter` | 8 scene summaries | Scene summary texts |
| `arc` | 4 chapter summaries | Chapter summary texts |

- Deterministic Pinecone id: `{tier}_{startSeq}_{endSeq}` in `sum_{instanceId}`
- Fetch children **by event range** (not id) so rebuilds are safe
- Stale + rebuild on event edit via `staleSummariesCoveringEvent`

---

## Worker libraries

| File | Role |
|------|------|
| `character-codex-extractor.ts` | LLM → per-turn codex deltas; kin-epithets resolve to existing role cards |
| `metadata-extractor.ts` | Post-prose scene metadata: KNOWN CAST/PLACES rosters, `viewpoint_moved`, `characters_departed`, protagonist-anchored choices |
| `structured-output.ts` | Parse/sanitize generation JSON + choices |
| `codex-compactor.ts` | Async shrink of long immutable fact lists |
| `nsfw-classifier.ts` | Lexicon cache (30 min refresh) for model routing |

---

## Dead letter & failure

Failed generation jobs after retries → `dead_letter_jobs` collection + WS `generation_failed`. Generation lock always released.

---

## Verification

| Command | Purpose |
|---------|---------|
| `bun run scripts/rewind-audit.ts` | Integration: clone instance, rewind, 27+ projection assertions |
| `bun run audit:choices` | Live LLM audit: first-person chips, canonical presence names |
| `bun run audit:location` | Live audit: phantom travel, returns reuse KNOWN PLACES, presence fold |
| `bun run audit:codex-dedup` | Live audit: kin-epithet resolves to existing codex card |
| `bun run audit:replay-edit` | Integration: replay + edit + variant select restore per-variant chips |
| `bun run merge:character` | Manual repair: merge duplicate codex cards (`<instanceId> "<keep>" "<dupe>"`) |
| `GET /admin/instances/:id/continuity-audit` | On-demand drift audit |
