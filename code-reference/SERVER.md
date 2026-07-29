# Server Code Reference

`everlore-server/` — production TypeScript grouped by folder. Aligns with [API.md](../server/API.md), [SERVICES.md](../server/SERVICES.md), [WORKERS.md](../server/WORKERS.md), [BILLING.md](../server/BILLING.md), [DATA_MODEL.md](../server/DATA_MODEL.md).

---

## Hot path (read these first)

Understanding one player turn:

```text
play-ws.service.ts
  → billing.service.ts (reserve Ink)
  → generation.service.ts (enqueue)
  → worker/generation.processor.ts
       → context-packet.service.ts
       → prompt-builder.ts
       → ai/client.ts (prose stream — not structured JSON turn body)
       → post-prose: metadata, deterministic signals, entity adjudicator,
         kinship / relation-candidates, signal_ledger, …
       → worker settles / releases billing reservation
       → worker/memory.processor.ts (async)
```

Memory & rewind: `memory.service.ts`, `entity-graph.service.ts`, `kinship-graph.service.ts`, `rag.provider.ts`

---

## `src/config/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `env.ts` | Load/validate environment variables | Singleton at import |
| `mongo.ts` | Mongo client + typed `mongoColl` accessors | Lazy connect; runs index reconcile once |
| `mongo-indexes.ts` | All collection indexes; drop deprecated | Batch at startup |
| `openai.ts` | OpenAI + OpenRouter SDK singletons | Client reuse |
| `pinecone.ts` | Pinecone client singleton | Client reuse |
| `redis.ts` | General, pub/sub, and BullMQ Redis clients | Separate connections for pub/sub vs queue |

---

## `src/controllers/` & `src/routes/`

Thin HTTP/WS handlers — auth check, parse input, call service, set status.

| Controller | Routes file | Prefix |
|------------|-------------|--------|
| `auth.controller.ts` | `auth.routes.ts` | `/auth` |
| `template.controller.ts` | `template.routes.ts` | `/templates` |
| `instance.controller.ts` | `instance.routes.ts` | `/instances` |
| `persona.controller.ts` | `persona.routes.ts` | `/personas` |
| `chronicle.controller.ts` | `chronicle.routes.ts` | `/chronicle` (Lore Tome + kinship + relation-candidates + edit/rewind/track) |
| `billing.controller.ts` | `billing.routes.ts` | `/billing` |
| `play-ws.controller.ts` | `ws.routes.ts` | `/ws/play` |
| `admin.controller.ts` | `admin.routes.ts` | `/admin` |

---

## `src/services/`

### Gameplay & memory

| File | Purpose | Optimizations |
|------|---------|---------------|
| `generation.service.ts` | Enqueue generate/replay/side_chat/world_action; `loadInstance` bootstrap | **Thin dispatch** — no RAG here; queue retention |
| `play-ws.service.ts` | WS registry, Redis pub/sub, locks, rate limits, **Ink reserve** before enqueue | In-memory socket map; generation lock |
| `billing.service.ts` | Wallet, catalog, reserve/settle/release, Google verify, simulate, RTDN sync | Ledger-backed balance; idempotent settle/release |
| `google-play.service.ts` | Play Billing API verify / acknowledge / consume / RTDN bearer | Called only by billing service |
| `context-packet.service.ts` | Build `ContextPacket` / `SideChatPacket` in worker | **`Promise.all`** RAG + summaries; codex pool; memory-driven pins |
| `memory.service.ts` | Chronicle CRUD, recap, threads, rewind, replay, edit propagation | Per-variant **choices/presence**; projection on mutation |
| `character-codex.service.ts` | Codex deltas, ranking, protagonist seed, ledger replay | Recency ranking; fact caps; async compaction |
| `entity-graph.service.ts` | Entities, edges, location facts, mention match, rewind repair | Name registry; provenance caps |
| `kinship-graph.service.ts` | Structural family graph (`entity_edges` type `kinship`) | Inverse-close, co-parent derivation, ledger rebuild |
| `relation-candidate.service.ts` | Player review queue (not canon until accept) | `propose` / `listOpen` / `resolve` |
| `memory-supersession.service.ts` | Archive vectors when codex retires facts; version-link marks | Pinecone match scoped MAIN origin |

### Retrieval & place/time

| File | Purpose | Optimizations |
|------|---------|---------------|
| `../providers/rag.provider.ts` | Hybrid RAG + summary search | **RRF fusion**; timeline filter; fail-closed side-chat gate |
| `time.service.ts` | Calendars, branches, anchors, flashback edit | Parallel calendar listing |
| `location.service.ts` | Places list + place journal API | Aggregation limits |
| `side-chat.service.ts` | Side-chat thread list/history | Aggregation pipeline |
| `projection-checkpoint.service.ts` | Chunked world projection checkpoints | Cron fan-out |

### Lifecycle & ops

| File | Purpose | Optimizations |
|------|---------|---------------|
| `instance.service.ts` | Instance CRUD, **Redis session cache**, settings | `session:{id}` TTL; bust on mutation |
| `template.service.ts` | Template CRUD, publish, lore embed | Pinecone upsert on publish |
| `deletion.service.ts` | Cascade delete/reset | Batch Mongo + namespace delete |
| `persona.service.ts` | Persona CRUD | Parallel session bust |
| `continuity-audit.service.ts` | Read-only cross-checks | Parallel collection reads |
| `admin.service.ts` | Admin dashboard CRUD, drift listing, projections | `Promise.all` counts |
| `auth.service.ts` | Users, JWT, Argon2 | Tier refresh on HTTP auth plugin |
| `autofill.service.ts` | Forge one-shot LLM draft | Single JSON LLM call |
| `image.service.ts` / `storage.service.ts` | Image gen → S3 | CDN cache headers |
| `pinecone-cleanup.service.ts` | Vector/namespace delete helpers | Best-effort |
| `template-cast.service.ts` | Authored cast extraction helpers | — |

---

## `src/models/`

TypeScript interfaces only — no runtime logic. Key docs:

| File | Represents |
|------|------------|
| `world-event.model.ts` | Event ledger (main + side_chat + travel + codex deltas) |
| `memory.model.ts` | Memory atom + entity ids + version links + side-chat scope |
| `character-profile.model.ts` | Codex card + relationship meters |
| `entity.model.ts` / `entity-edge.model.ts` | Graph nodes and edges (incl. kinship) |
| `world-instance.model.ts` | Play session + time/location cursors + meta |
| `scene/chapter/arc-summary.model.ts` | Summary tiers |
| `time.model.ts` | Calendar + TimeAnchor + timeline branch |
| `billing.model.ts` | Ink ledger, entitlements, store purchases |
| `projection.model.ts` | Shared active/stale/superseded/archived status |
| `collections.ts` | Collection name constants |

Also in data model (see DATA_MODEL.md): `relation_candidates`, `signal_ledger`, projection anomalies.

---

## `src/utils/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `prompt-builder.ts` | LLM messages from context packet | **Static cacheable prefix**; per-section **token budgets**; **1000-token recents floor** |
| `prose-hygiene.ts` | Validate/repair quote/italic/POV | Rules-first; LLM repair only on failure |
| `token-counter.ts` | Tiktoken count | Lazy encoder singleton |
| `event-window.ts` | Shared limits: 6 / 20 / 50 | Constants |
| `narrative-styles.ts` | Voice presets + length → max tokens | Static presets |
| `chat-modes.ts` | Flow mode directives | Static |
| `player-input-parser.ts` | Split spoken vs `*narration*` | Pure function |
| `state-mutator.ts` | Apply stat/flag mutations | Pure |
| `mongo-id.ts` | ObjectId parse/stringify | — |
| `logger.ts` | Structured logs | — |

---

## `src/ai/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `client.ts` | `callLLM`, `callLLMStream` | Stream idle timeout; abort; OpenRouter routing |
| `models.ts` | Central model ID map | Env overrides |
| `embedding.ts` | `embed`, **`embedBatch`** | Batch for maintenance dedup |
| `image.ts` / `tts.ts` | Image gen / speech | Optional abort |

---

## `worker/processors/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `generation.processor.ts` | **Main turn** — packet → **prose stream** → post-prose pipeline → event → follow-ups → **billing settle** | Fire-and-forget graph/kinship/candidates/signal_ledger; 12-turn summary blocks |
| `side-chat.processor.ts` | Private NPC chat | No main time/location advance; scoped packet |
| `memory.processor.ts` | Post-turn memory extract + embed; materialize `updates_memory_ids` | 0–3 atoms; entity resolve |
| `summary.processor.ts` | Scene → chapter → arc rollups | Deterministic vector ids |
| `replay.processor.ts` | Stream replay variants | Lock release in `finally` |
| `maintenance.processor.ts` | Decay, dedup, repair, drift audit, checkpoints, memory-link repair | Pinecone fetch batches |

---

## `worker/lib/`

| File | Purpose |
|------|---------|
| `prose-stream-filter.ts` | Strip accidental JSON envelopes from player-facing stream |
| `choice-tail.ts` | Parse/filter narrator `==CHOICES==` tail off the prose stream |
| `structured-output.ts` | Choice/JSON helper types + sanitizers (not the live turn response format) |
| `metadata-extractor.ts` | Post-prose witness: presence, place, time, choices |
| `entity-adjudicator.ts` | Semantic judge for strong person-candidate terms |
| `presence-gap-detector.ts` | Prose people missing from presence/cards |
| `movement-signal.ts` / `time-skip-signal.ts` / `party-signal.ts` / `scene-location-signal.ts` | Deterministic backstops |
| `kinship-*.ts` / `premise-kinship-extractor.ts` | Kinship assertions, transitions, hygiene, epithets |
| `relation-candidate-detector.ts` / `canon-revision-detector.ts` | Narrator review queue proposals |
| `signal-ledger.ts` | FP/FN tallies for `signal_ledger` |
| `projection-anomaly-detector.ts` | Prose vs projection findings |
| `character-codex-extractor.ts` / `codex-compactor.ts` | Codex deltas + async shrink |
| `choice-grounding.ts` | Drop ungrounded chips |
| `nsfw-classifier.ts` | Route to NSFW model; lexicon cache |
| `side-chat-privacy.ts` / `side-chat-reachability.ts` | Fail-closed deltas + reachability framing |

---

## `worker/index.ts`

Starts BullMQ workers + registers cron: decay, dedup, summary-repair, continuity-audit, projection-checkpoint. Warms NSFW lexicon on boot. Owns **billing settle/release** on job completion / failure paths — see [WORKERS.md](../server/WORKERS.md).

---

## Optimizations summary table

| Technique | Benefit | Risk if removed |
|-----------|---------|-----------------|
| Context packet in worker | Retrieval before codex pins | Dormant characters miss canon |
| Prose stream + choice tail | Fast TTFT; chips without JSON envelope | Tail parse bugs drop chips |
| Token budgets + recents floor | Prompt can't starve last turns | Model loses thread at scale |
| Redis session cache | Faster WS turns | More Mongo reads |
| Ink reserve → settle/release | No free turns; no charge on pre-stream fail | Balance drift if settle skipped |
| Fire-and-forget post-turn | Faster stream complete | Slightly stale graph/kinship for ~1s |
| Hybrid RAG + RRF | Name + meaning recall | Pure vector misses exact names |
| Side-chat fail-closed gate | Privacy | Secret leaks to main story |
