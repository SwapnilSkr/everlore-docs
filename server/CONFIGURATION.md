# Everlore Server — Configuration

All runtime configuration is loaded from environment variables via [`everlore-server/src/config/env.ts`](../../everlore-server/src/config/env.ts). Bun loads `.env` automatically. Copy [`.env.example`](../../everlore-server/.env.example) to `.env`.

Missing **required** vars only log a warning and fall back to unsafe local defaults — do **not** rely on that in production.

Related: [DEPLOYMENT.md](./DEPLOYMENT.md) · [BILLING.md](./BILLING.md) · [SECURITY.md](./SECURITY.md) · [SFW_MODELS.md](./SFW_MODELS.md) · [NSFW_MODELS.md](./NSFW_MODELS.md)

---

## Required (production)

| Variable | Description | Example |
|----------|-------------|---------|
| `JWT_SECRET` | Symmetric JWT sign/verify secret | `openssl rand -hex 32` |
| `MONGODB_URI` | MongoDB connection (db name in path) | `mongodb+srv://…/everlore?retryWrites=true&w=majority` |
| `REDIS_URL` | Redis for BullMQ, pub/sub, locks, rate limits | `redis://127.0.0.1:6379` or `rediss://…` |
| `OPENAI_API_KEY` | Embeddings (+ OpenAI-routed aux models) | `sk-…` |
| `PINECONE_API_KEY` | Vector RAG / memory upsert | `pcsk_…` |

---

## Server & CORS

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | HTTP + WebSocket listen port |
| `CLIENT_ORIGINS` | `http://localhost:3000,http://localhost:8080` | Comma-separated CORS origins |

---

## Pinecone

| Variable | Default | Description |
|----------|---------|-------------|
| `PINECONE_INDEX` | `nexus-memories` | Index name |
| Namespaces (not env) | — | `lore_{templateId}`, `mem_{instanceId}`, `sum_{instanceId}` |

---

## LLM providers & models

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENROUTER_API_KEY` | `` | Narration, images, TTS, most chat models |
| `OPENROUTER_PREFERRED_P90_LATENCY_SECONDS` | `3` | Provider latency preference; `0` disables |
| `NARRATION_SFW_MODEL` | `deepseek/deepseek-v3.2` | Default SFW narrator (prose-only) |
| `NARRATION_NSFW_MODEL` | `gryphe/mythomax-l2-13b` | Default NSFW narrator |
| `IMAGE_MODEL` | `bytedance-seed/seedream-4.5` | Cover/avatar/background generation |
| `AUTHORING_MODEL` | `google/gemini-2.5-flash-lite` | World/character autofill drafts |
| `TTS_MODEL` | `hexgrad/kokoro-82m` | OpenRouter speech (eval/scripts; no product route yet) |
| `CHEAP_RANK_MODEL` | `gpt-4o-mini` | Aux model for optional re-rank / NSFW intent |
| `RAG_RERANK_ENABLED` | `false` | LLM re-rank of fused RAG pool (adds TTFT — keep off in prod unless intentional) |
| `NSFW_INTENT_DEFER_ENABLED` | `false` | Borderline lexicon scores → post-stream intent judge; arms next turn |

Templates may override narration models via `model_preferences`. See model docs for candidates and testing scripts.

---

## Auth

| Variable | Default | Description |
|----------|---------|-------------|
| `GOOGLE_CLIENT_ID` | `` | Web OAuth client ID; JWT audience for Google ID tokens |
| `TWILIO_ACCOUNT_SID` | `` | Use `AC_MOCK_SID` for local mock OTP (`123456`) |
| `TWILIO_AUTH_TOKEN` | `` | Twilio auth token |
| `TWILIO_VERIFY_SERVICE_SID` | `` | Twilio Verify service SID |
| `DISABLE_OTP_RATE_LIMIT` | `false` | Skip OTP Redis rate limits — **dev only**; must be `false` in production |

Flutter must request a Google ID token for the same `GOOGLE_CLIENT_ID`. Phones must be E.164 (e.g. `+15551234567`).

---

## Admin suite

| Variable | Default | Description |
|----------|---------|-------------|
| `ADMIN_USERNAME` | `` | HTTP Basic username for `/admin/*` |
| `ADMIN_PASSWORD` | `` | HTTP Basic password |

If either is unset, `/admin/*` returns **503** (`Admin credentials are not configured`). Comparison is constant-time hashed. See [SECURITY.md](./SECURITY.md).

---

## Throughput & rate limits

| Variable | Default | Where applied | Meaning |
|----------|---------|---------------|---------|
| `GENERATION_CONCURRENCY` | `3` | `worker/index.ts` BullMQ Worker | Simultaneous story turns **per worker process** |
| `GENERATION_RATE_MAX` | `10` | Same Worker `limiter` / 60s | Max new generation jobs started per minute **per worker** |
| `CHAT_RATE_MAX` | `10` | `middleware/rate-limit.ts` `chat` | Max chat/continue/side_chat/replay actions **per user** / 60s |
| `TEMPLATE_CREATE_RATE_MAX` | `5` | `template_create` | Max world creates **per user** / 24h |

### Fixed (non-env) API rate limits

| Action | Max | Window |
|--------|-----|--------|
| `memory_edit` | 30 | 1 hour |
| `image_generate` | 40 | 1 hour |
| `autofill` | 30 | 1 hour |
| `auth_attempt` | 10 | 5 minutes |
| `otp_send` | 5 | 10 minutes |
| `otp_verify` | 10 | 10 minutes |

Worker-side (hardcoded in `worker/index.ts`): `memory-curation` concurrency 5 / 20 per min; `scene-summary` concurrency 2; `maintenance` concurrency 1.

See [DEPLOYMENT.md](./DEPLOYMENT.md) for how concurrency affects multi-user capacity.

---

## Billing / Story Ink / Google Play

| Variable | Default | Description |
|----------|---------|-------------|
| `BILLING_ENFORCEMENT_ENABLED` | `false` | When `true`, insufficient Ink blocks billable actions |
| `BILLING_SIMULATION_ENABLED` | `false` | Simulated checkout for local/QA — **hard-blocked if `NODE_ENV=production`** |
| `GOOGLE_PLAY_PACKAGE_NAME` | `` | Android package name |
| `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON` | `` | Full service-account JSON string (Publisher API) |
| `GOOGLE_PLAY_RTDN_AUDIENCE` | `` | OIDC audience for RTDN push (usually the RTDN URL) |
| `GOOGLE_PLAY_RTDN_SERVICE_ACCOUNT_EMAIL` | `` | Expected push sender email |

Full product IDs, RTDN setup, and test checklist: [BILLING.md](./BILLING.md).

---

## AWS S3 + CloudFront (media)

Parsed by `env.ts`:

| Variable | Default | Description |
|----------|---------|-------------|
| `AWS_REGION` | `ap-south-1` | Bucket region |
| `S3_BUCKET` | `` | Private media bucket |
| `CDN_BASE_URL` | `` | CloudFront base URL (no trailing slash) |

Read by the AWS SDK (not listed in `env.ts` interface, but required for uploads):

| Variable | Description |
|----------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM access key |
| `AWS_SECRET_ACCESS_KEY` | IAM secret |

Serve media **only** via CDN (OAC). Seed fallback cover: `bun run seed:default-cover` → `<CDN_BASE_URL>/media/defaults/everlore-cover.webp`.

---

## Production checklist

```text
JWT_SECRET=<long random>
MONGODB_URI=<Atlas URI with /everlore>
REDIS_URL=<local or managed>
OPENAI_API_KEY=…
PINECONE_API_KEY=…
OPENROUTER_API_KEY=…          # required for default narration + images
CLIENT_ORIGINS=https://app.example.com
GOOGLE_CLIENT_ID=…
DISABLE_OTP_RATE_LIMIT=false
ADMIN_USERNAME=…              # or leave unset to disable /admin
ADMIN_PASSWORD=…
BILLING_ENFORCEMENT_ENABLED=false   # flip true only after Play setup
BILLING_SIMULATION_ENABLED=false
S3_BUCKET=… CDN_BASE_URL=… AWS_* =…
GENERATION_CONCURRENCY=2      # tune for box size
```

---

## Local development tips

- Mongo: `mongodb://localhost:27017/everlore`
- Redis: `redis://localhost:6379`
- Twilio mock: `TWILIO_ACCOUNT_SID=AC_MOCK_SID` (OTP `123456`)
- `DISABLE_OTP_RATE_LIMIT=true` is convenient locally; never ship that to prod
- Simulated billing (non-production only): see [BILLING.md](./BILLING.md)

---

## Source of truth

| File | Role |
|------|------|
| `everlore-server/src/config/env.ts` | Parsed vars + defaults |
| `everlore-server/.env.example` | Template for operators |
| `everlore-server/src/middleware/rate-limit.ts` | Rate-limit tables |
| `everlore-server/worker/index.ts` | Worker concurrency / limiters |
