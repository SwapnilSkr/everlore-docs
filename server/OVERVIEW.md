# Everlore Server — Architecture Overview

Everlore is an AI-powered narrative roleplay platform: players engage persistent AI worlds over HTTP and WebSocket. The backend is Bun + Elysia (API) and a separate BullMQ worker cluster (generation, memory, summaries, maintenance). Content can be SFW or NSFW with lexicon-based routing and optional deferred intent judging.

**Related docs:** [API.md](./API.md) · [WORKERS.md](./WORKERS.md) · [BILLING.md](./BILLING.md) · [CONFIGURATION.md](./CONFIGURATION.md) · [DEPLOYMENT.md](./DEPLOYMENT.md) · [DATA_MODEL.md](./DATA_MODEL.md) · [SERVICES.md](./SERVICES.md)

---

## High-level architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLIENT (Flutter / web)                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                          HTTP + WS /ws/play?token=…
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              API SERVER (Elysia) — routes → controllers → services            │
│  /auth  /templates  /instances  /chronicle  /personas  /billing  /admin      │
│  WS /ws/play                                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │   MongoDB   │      │    Redis    │      │  Pinecone   │
    │  (primary)  │      │ cache/queue │      │  (vectors)  │
    │             │      │ pub/sub WS  │      │             │
    └─────────────┘      └─────────────┘      └─────────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │   BullMQ queues     │
                     │ generation · memory │
                     │ scene-summary ·     │
                     │ maintenance         │
                     └─────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WORKER CLUSTER                                       │
│  Generation (conc: env) · Memory (5) · Summary (2) · Maintenance (1)         │
└─────────────────────────────────────────────────────────────────────────────┘
           │                              │
           ▼                              ▼
  OpenRouter / OpenAI LLMs      S3 (private) → CloudFront CDN (media)
  DeepSeek V3.2 (SFW prose)     Story Ink ledger + Google Play billing
  MythoMax (NSFW prose)
  gpt-4o-mini (metadata, etc.)
```

---

## HTTP & WebSocket routes

Registered in `src/index.ts`. Path prefixes:

| Prefix | Module | Role |
|--------|--------|------|
| `/auth` | `auth.routes.ts` | Register, login, Google, phone OTP |
| `/templates` | `template.routes.ts` | Browse, create, publish world templates |
| `/instances` | `instance.routes.ts` | Start / manage play instances |
| `/chronicle` | `chronicle.routes.ts` | Lore Tome surfaces (events, codex, calendar, …) |
| `/personas` | `persona.routes.ts` | Player personas |
| `/billing` | `billing.routes.ts` | Story Ink wallet, catalog, Play verify / RTDN |
| `/admin` | `admin.routes.ts` | Ops / diagnostics (auth-gated) |
| `WS /ws/play` | `ws.routes.ts` | Real-time play (chat, continue, side chat, replay) |

Canonical request/response shapes: [API.md](./API.md) · [SCHEMAS.md](./SCHEMAS.md). Auth details: [SECURITY.md](./SECURITY.md).

Layout: `routes/` register paths and schemas; `controllers/` orchestrate (rate limits, tier checks); `services/` hold domain logic. WS play uses `play-ws` controller + service. See [SERVICES.md](./SERVICES.md).

---

## MongoDB collections

Canonical names live in `everlore-server/src/models/collections.ts`. Full field docs: [DATA_MODEL.md](./DATA_MODEL.md).

| Collection | Purpose |
|------------|---------|
| `users` | Accounts, preferences, NSFW opt-in |
| `world_templates` | World blueprints |
| `world_instances` | Active play sessions |
| `events` | Turn / chronicle event history |
| `memories` | Long-term memory docs |
| `scene_summaries` / `chapter_summaries` / `arc_summaries` | Compressed narrative layers |
| `characters` | Character codex cards |
| `entities` / `entity_edges` | Place/person graph + kinship edges |
| `story_calendars` / `timeline_branches` | In-world time |
| `personas` | Player personas |
| `dead_letter_jobs` | Failed job diagnostics |
| `generation_logs` | Per-turn model / token telemetry |
| `nsfw_lexicon` | NSFW routing word list |
| `projection_anomalies` / `signal_ledger` | Projection / signal bookkeeping |
| `projection_checkpoints` / `projection_checkpoint_chunks` | Rebuildable projection snapshots |
| `location_stats` | Materialized place stats |
| `relation_candidates` | Pending relationship promotions |
| `ink_ledger` | Story Ink balance mutations (authoritative) |
| `billing_entitlements` | Subscription / tier entitlements |
| `store_purchases` | Google Play purchase records |

---

## Narration pipeline (prose stream + post metadata)

Narration is **not** a single structured JSON-Schema turn response.

1. **Classify & route** (sync, lexicon / Ardent mode — no LLM on the TTFT path for Free Play).
2. **Stream prose** via `callLLMStream` + `makeProseStreamFilter` — plain narrative text token-by-token to the client over Redis pub/sub → WS.
3. **After the stream**, a separate **metadata extract** pass (`extractSceneMetadata` / `AI_MODELS.metadata`, default `gpt-4o-mini`) derives scene tag, tone, choices, state/flag mutations, presence, location hints, etc.
4. Persist the event, update session/world state, enqueue memory/summary work, settle Story Ink, publish `generation_complete`.

Optional narrator choice-tail (`NARRATOR_CHOICES=on`) can emit tap-to-play options in a sentinel-delimited prose suffix; otherwise choices come from the metadata pass. Details: [WORKERS.md](./WORKERS.md), [system-guide/02-one-turn-journey.md](../system-guide/02-one-turn-journey.md).

### Default models

| Role | Default | Provider | Override |
|------|---------|----------|----------|
| SFW narration | `deepseek/deepseek-v3.2` | OpenRouter | `NARRATION_SFW_MODEL` |
| NSFW narration | `gryphe/mythomax-l2-13b` | OpenRouter | `NARRATION_NSFW_MODEL` |
| Metadata / bookkeeping | `gpt-4o-mini` | OpenAI | `MODEL_METADATA` |

Source of truth: `src/ai/models.ts`. Model catalogs: [SFW_MODELS.md](./SFW_MODELS.md) · [NSFW_MODELS.md](./NSFW_MODELS.md).

---

## Media: S3 + CloudFront

Generated images (covers, avatars, chat backgrounds) upload to a **private S3 bucket** and are served via **CloudFront** (`CDN_BASE_URL`). See `src/services/storage.service.ts` and [CONFIGURATION.md](./CONFIGURATION.md) (`S3_BUCKET`, `CDN_BASE_URL`, AWS credentials).

---

## Billing: Story Ink

Spendable currency for story turns, autofill, and image previews. Ledger in Mongo (`ink_ledger`) is authoritative; Google Play sells subscriptions and Ink packs. Enforcement is opt-in outside production (`BILLING_ENFORCEMENT_ENABLED=true`); production also auto-enforces once Google Play credentials are configured.

Full product/catalog/reserve-settle flow: **[BILLING.md](./BILLING.md)**. Play Console setup: `everlore-server/BILLING_PLAY_SETUP.md` (points here as canonical).

---

## Redis

| Use | Notes |
|-----|--------|
| Session / locks | Generation locks with heartbeat; session cache |
| Rate limits | Auth OTP, generation, etc. |
| BullMQ | Queue backend |
| **Pub/sub for WS** | Workers `PUBLISH` to `user:{userId}:events`; API `play-ws` subscriber fans out frames to connected clients |

---

## Pinecone

| Namespace pattern | Contents |
|-------------------|----------|
| `lore_{templateId}` | Template lore chunks |
| `mem_{instanceId}` | Instance memories |

Embeddings: OpenAI `text-embedding-3-small` (`AI_MODELS.embedding`).

---

## Queues & workers

| Queue | Concurrency | Role |
|-------|-------------|------|
| `generation` | `GENERATION_CONCURRENCY` (default 3) | Main turn, side chat, replay |
| `memory-curation` | 5 | Extract & store memories |
| `scene-summary` | 2 | Compress scene history |
| `maintenance` | 1 | Decay, dedup, summary repair, audits, checkpoints |

See [WORKERS.md](./WORKERS.md). Deploy / scale: [DEPLOYMENT.md](./DEPLOYMENT.md).

---

## Request flow (chat turn)

```
1. Client → WS /ws/play (chat / continue / …)
2. API auth, rate limit, Story Ink reserve (if enforcement on)
3. Redis generation lock + enqueue BullMQ generation job
4. Worker: RAG (Pinecone) + prompt build
5. Worker: stream prose → Redis pub/sub → API → client deltas
6. Worker: metadata extract (separate LLM call)
7. Worker: persist event, update Mongo/session, optional NSFW intent arming
8. Worker: settle/release Ink; enqueue memory/summary as needed
9. Worker: publish generation_complete (and related frames)
10. Client renders complete turn
```

---

## Key design decisions

1. **API vs workers** — I/O-bound API; CPU/LLM work in workers; independent scale.
2. **Prose stream + post metadata** — TTFT stays low; bookkeeping does not block first token.
3. **NSFW routing** — Mature world + user opt-in + (Ardent mode **or** lexicon/`nsfw_intent` momentum). Borderline clean-language intent can be judged **after** stream when `NSFW_INTENT_DEFER_ENABLED` — see [NSFW_MODELS.md](./NSFW_MODELS.md).
4. **Story Ink** — Reserve before enqueue; settle/release in the worker lifecycle.
5. **Projections** — Codices, kinship, locations rebuild from event ledgers + checkpoints.

---

## Technology stack

| Layer | Technology |
|-------|------------|
| Runtime | Bun |
| API | Elysia (+ JWT, CORS, TypeBox) |
| Identity | Google tokeninfo, Twilio Verify OTP, Argon2 passwords |
| DB | MongoDB |
| Cache / queue / WS fanout | Redis (ioredis) + BullMQ |
| Vectors | Pinecone |
| LLMs | OpenRouter (narration defaults) + OpenAI (metadata/embeddings) |
| Media | AWS S3 + CloudFront |
| Billing | Story Ink ledger + Google Play Developer API |

Env reference: [CONFIGURATION.md](./CONFIGURATION.md). Cheap AWS prod plan: [DEPLOYMENT.md](./DEPLOYMENT.md).
