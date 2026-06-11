# Server Code Reference

`everlore-server/` — every production TypeScript file, grouped by folder.

---

## Hot path (read these first)

Understanding one player turn:

```text
play-ws.service.ts
  → generation.service.ts (enqueue)
  → worker/generation.processor.ts
       → context-packet.service.ts
       → prompt-builder.ts
       → ai/client.ts (stream)
       → worker/memory.processor.ts (async)
```

Memory & rewind: `memory.service.ts`, `entity-graph.service.ts`, `rag.provider.ts`

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

## `src/controllers/`

Thin HTTP/WS handlers — auth check, parse input, call service, set status.

| File | Routes |
|------|--------|
| `auth.controller.ts` | `/auth/*` |
| `template.controller.ts` | `/templates/*` |
| `instance.controller.ts` | `/instances/*` |
| `persona.controller.ts` | `/personas/*` |
| `chronicle.controller.ts` | `/chronicle/*` (all Lore Tome + edit/rewind APIs) |
| `play-ws.controller.ts` | WebSocket `/ws/play` |
| `admin.controller.ts` | `/admin/*` + continuity audit |

---

## `src/routes/`

Elysia route trees — TypeBox schemas, auth plugins, delegate to controllers.

| File | Prefix |
|------|--------|
| `auth.routes.ts` | `/auth` |
| `template.routes.ts` | `/templates` |
| `instance.routes.ts` | `/instances` |
| `persona.routes.ts` | `/personas` |
| `chronicle.routes.ts` | `/chronicle` |
| `admin.routes.ts` | `/admin` |
| `ws.routes.ts` | `/ws/play` |

---

## `src/services/`

### Gameplay & memory

| File | Purpose | Optimizations |
|------|---------|---------------|
| `generation.service.ts` | Enqueue generate/replay/side_chat; `loadInstance` bootstrap | **Thin dispatch** — no RAG here; fresh NSFW read from DB; queue retention |
| `play-ws.service.ts` | WS registry, Redis pub/sub, locks, rate limits | In-memory socket map; 240s generation lock |
| `context-packet.service.ts` | Build `ContextPacket` / `SideChatPacket` in worker | **`Promise.all`** RAG + summaries; codex pool 40→16; memory-driven pins; event **projection** |
| `memory.service.ts` | Chronicle CRUD, recap, threads, rewind, replay, edit propagation | Per-variant **choices/presence** on replay/edit/select; heavy **projection**; text search |
| `character-codex.service.ts` | Codex deltas, ranking, protagonist seed, ledger replay | Recency ranking formula; fact caps; async compaction trigger |
| `entity-graph.service.ts` | Entities, edges, location facts, mention match, rewind repair | Name registry scan; provenance caps on edges |
| `memory-supersession.service.ts` | Archive vectors when codex retires facts | Pinecone similarity search per retired fact (high threshold 0.82) |

### Retrieval & place/time

| File | Purpose | Optimizations |
|------|---------|---------------|
| `../providers/rag.provider.ts` | Hybrid RAG + summary search | **RRF fusion**; parallel arms; timeline filter; fail-closed side-chat gate; access_count bump |
| `time.service.ts` | Calendars, branches, anchors, flashback edit | Parallel calendar listing; projected event reads |
| `location.service.ts` | Places list + place journal API | **`Promise.all`** aggregations; limits on journal size |
| `side-chat.service.ts` | Side-chat thread list/history | Aggregation pipeline; projected character card |

### Lifecycle & ops

| File | Purpose | Optimizations |
|------|---------|---------------|
| `instance.service.ts` | Instance CRUD, **Redis session cache**, settings | `session:{id}` TTL **3600s**; bust on mutation |
| `template.service.ts` | Template CRUD, publish, lore embed | Pinecone upsert on publish |
| `deletion.service.ts` | Cascade delete/reset | Batch Mongo + namespace delete |
| `persona.service.ts` | Persona CRUD | Parallel session bust for affected instances |
| `continuity-audit.service.ts` | 8-check read-only audit | Parallel collection reads |
| `admin.service.ts` | Admin dashboard CRUD, drift listing | `Promise.all` counts; password projection excluded |
| `auth.service.ts` | Users, JWT, Argon2 | — |
| `autofill.service.ts` | Forge one-shot LLM draft | Single JSON LLM call |
| `image.service.ts` / `storage.service.ts` | Image gen → S3 | CDN cache headers |
| `pinecone-cleanup.service.ts` | Vector/namespace delete helpers | Best-effort |

---

## `src/models/`

TypeScript interfaces only — no runtime logic. Key docs:

| File | Represents |
|------|------------|
| `world-event.model.ts` | Event ledger (main + side_chat + travel + codex deltas) |
| `memory.model.ts` | Memory atom + entity ids + side-chat scope |
| `character-profile.model.ts` | Codex card + relationship meters |
| `entity.model.ts` / `entity-edge.model.ts` | Graph nodes and edges |
| `world-instance.model.ts` | Play session + time/location cursors + meta |
| `scene/chapter/arc-summary.model.ts` | Summary tiers |
| `time.model.ts` | Calendar + TimeAnchor + timeline branch |
| `projection.model.ts` | Shared active/stale/superseded/archived status |
| `collections.ts` | Collection name constants |

Full schema narrative: [../server/DATA_MODEL.md](../server/DATA_MODEL.md)

---

## `src/utils/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `prompt-builder.ts` | LLM messages from context packet | **Static cacheable prefix**; per-section **token budgets**; **1000-token recents floor** |
| `prose-hygiene.ts` | Validate/repair quote/italic/POV | Rules-first; LLM repair only on failure |
| `token-counter.ts` | Tiktoken count | Lazy encoder singleton |
| `event-window.ts` | Shared limits: 6 / 20 / 50 | Constants |
| `narrative-styles.ts` | Voice presets + length → max tokens | Static presets in cacheable block |
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
| `generation.processor.ts` | **Main turn** — packet → prompt → stream → event → follow-ups | **Fire-and-forget**: location facts, codex compaction, supersession, gen logs; async codex IIFE; travel detected server-side; 12-turn summary blocks |
| `side-chat.processor.ts` | Private NPC chat | No main time/location advance; scoped packet |
| `memory.processor.ts` | Post-turn memory extract + embed | 0–3 atoms; entity resolve; Pinecone upsert |
| `summary.processor.ts` | Scene → chapter → arc rollups | Deterministic vector ids; range-keyed children |
| `replay.processor.ts` | Stream replay variants | Lock release in `finally` |
| `maintenance.processor.ts` | Decay, dedup, repair, drift audit | **Pinecone fetch 200/batch** (no re-embed); O(n²) dedup compare |

---

## `worker/lib/`

| File | Purpose |
|------|---------|
| `character-codex-extractor.ts` | LLM → codex deltas per turn |
| `metadata-extractor.ts` | Post-prose: location, time_elapsed, present_characters |
| `structured-output.ts` | Parse generation JSON + choice chips |
| `codex-compactor.ts` | Shrink long fact lists (async) |
| `nsfw-classifier.ts` | Route to NSFW model; **lexicon cache** 30 min |

---

## `worker/index.ts`

Starts 4 BullMQ workers + registers cron: decay, dedup, summary-repair, continuity-audit. Warms NSFW lexicon on boot.

---

## `scripts/` (dev/ops)

| Script | Purpose |
|--------|---------|
| `rewind-audit.ts` | Integration test: clone instance, rewind, 27+ assertions |
| `choice-audit.ts` | Live audit: choice POV + canonical presence (`audit:choices`) |
| `location-audit.ts` | Live audit: location cursor, travel gating, presence fold (`audit:location`) |
| `codex-dedup-audit.ts` | Live audit: kin-epithet dedup (`audit:codex-dedup`) |
| `replay-edit-audit.ts` | Integration: replay/edit variant chips (`audit:replay-edit`) |
| `merge-character-cards.ts` | Manual codex dedup repair (`merge:character`) |
| `test-sfw-model.ts` / `test-nsfw-model.ts` | Model A/B with TTFT metrics |
| `seed-nsfw-lexicon.ts` | Seed classifier terms |
| `clean-redis-jobs.ts` | Queue cleanup |

---

## Optimizations summary table

| Technique | Benefit | Risk if removed |
|-----------|---------|-----------------|
| Context packet in worker | Retrieval before codex pins | Dormant characters miss canon |
| Token budgets + recents floor | Prompt can't starve last turns | Model loses thread at scale |
| Redis session cache | Faster WS turns | More Mongo reads |
| Fire-and-forget post-turn | Faster stream complete | Slightly stale location/codex for ~1s |
| Hybrid RAG + RRF | Name + meaning recall | Pure vector misses exact names |
| Projected rewind load | Deep rewind stays fast | OOM on huge histories |
| Pinecone fetch dedup | Cheap weekly dedup | Re-embed cost explosion |
| Side-chat fail-closed gate | Privacy | Secret leaks to main story |
