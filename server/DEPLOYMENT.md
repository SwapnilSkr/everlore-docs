# Everlore — Cheap AWS Production Deployment

Broke-friendly production plan for the Bun API + BullMQ worker. Complements [INFRASTRUCTURE.md](./INFRASTRUCTURE.md) (architecture), [CONFIGURATION.md](./CONFIGURATION.md) (env vars), [BILLING.md](./BILLING.md), and [SECURITY.md](./SECURITY.md).

**Goal:** lowest cost, good enough performance, autoscaling path later — without Fargate, ALB, NAT Gateway, Route 53, or ElastiCache on day one.

---

## Verdict

**Day-1 (chosen):** one public EC2 (`t4g.small`) running **Redis + API + worker + Caddy**, Porkbun DNS `A` record → Elastic IP, MongoDB Atlas, existing S3/CloudFront.

**Avoid on day-1:** ALB, NAT Gateway, Route 53 hosted zone, ElastiCache, Fargate, Redis Cloud (until multi-host workers).

**Scale-up:** keep API on-demand; add worker Spot ASG. Then move Redis off the API box (Redis Cloud Essentials or a tiny dedicated Redis) and flip `REDIS_URL`.

```text
Clients (Flutter / web)
        │
        ▼
Porkbun DNS ──A──► Elastic IP ──► Caddy (TLS) ──► Bun API + WebSocket
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                   Redis (local)   Mongo Atlas   Pinecone
                         ▲
                         │ BullMQ + pub/sub
                         │
                   Bun Worker (same box day-1)
                         │
                         ▼
                   OpenAI / OpenRouter
                   S3 / CloudFront (media)
```

Worker publishes stream events to Redis pub/sub; every API process that holds the user’s socket receives them. **Sticky sessions are not required.**

---

## Why this shape (cost)

| Idea | Reality for a low-budget launch |
|------|----------------------------------|
| ALB + Spot ASG | ALB ~$16–22/mo idle. Needed for multi-API later; skip until then. |
| Fargate | Worse $/perf than a small EC2 for Bun. |
| ElastiCache | Expensive vs on-box Redis or Redis Cloud Essentials. |
| Redis Cloud day-1 | Valid, but not cheapest; defer until a second host needs shared Redis. |
| Route 53 | Unnecessary. Keep Porkbun DNS → EIP. |
| NAT Gateway | ~$32/mo. Use a **public subnet** + EIP instead. |
| Spot for API | Spot reclaim drops WebSockets mid-turn. On-demand for API; Spot for workers later. |

Outbound app bandwidth to S3/LLM is usually fine. Bill killers are managed networking (ALB/NAT), not the Bun processes.

---

## On-box Redis vs Redis Cloud

**Day-1 choice: on-box Redis** (cheaper and sufficient on a single machine).

| | On-box Redis | Redis Cloud |
|--|--------------|-------------|
| Extra cost | **$0** | Free tier too weak for real play; real prod ≈ **$5–7/mo** Essentials |
| Latency | Localhost (best for token pub/sub) | Cross-network (fine, not free) |
| Ops | AOF + `noeviction` + memory cap | Managed |
| EC2 reboot/deploy | Redis restarts with the box; short queue blip | Redis stays up |
| Scale workers to other VMs | Awkward (must expose Redis safely) | Natural shared URL |
| Performance | **Does not degrade** vs Cloud — usually faster | Extra RTT per Redis op |

**Move to Redis Cloud when:** you add a second machine (worker Spot ASG), Redis and generation concurrency fight for RAM, or you want Redis to survive every EC2 reboot without babysitting.

App change is only `REDIS_URL` (`redis://127.0.0.1:6379` → `rediss://…`). See `everlore-server/src/config/redis.ts` (three ioredis clients, `maxRetriesPerRequest: null` for BullMQ).

---

## Day-1 infrastructure

Target roughly **$5–15/mo** compute (plus existing Atlas / S3 / LLM).

1. **EC2** `t4g.small` (or `t4g.micro` if traffic is tiny) in **`ap-south-1`** (same region as S3).
2. **Public subnet + security group:** `80/443` from the world; SSH or SSM only for operators. Redis bound to **`127.0.0.1` only** (never public).
3. **Elastic IP** attached (free while attached).
4. **Redis 7** on-box: `appendonly yes`, `maxmemory-policy noeviction`, `maxmemory` sized for the instance (leave headroom for Bun).
5. **Two systemd units** (or Docker Compose): `everlore-api` (`bun run start`), `everlore-worker` (`bun run start:worker`).
6. **Caddy** reverse proxy: HTTPS via Let’s Encrypt; long idle / WebSocket timeouts for `/ws/play` (generation locks are minutes-scale).
7. **Porkbun:** `A`/`AAAA` for `api.` → EIP. No nameserver move. No Route 53.
8. **Atlas:** allowlist the EC2 Elastic IP.
9. **Secrets:** GitHub Actions → `/etc/everlore.env` (or SSM later). Never bake into images.

Optional later: Cloudflare free proxy for DDoS/TLS (requires pointing Porkbun NS at Cloudflare — still $0, not Route 53). Not required if Caddy + Let’s Encrypt is enough.

---

## Concurrency, rate limits, and “how many users?”

There is **no single hard “max concurrent users”** in the app. Connected players and generating players are limited differently.

### Knobs (app env vars — not Docker CPU settings)

Defined in `everlore-server/src/config/env.ts`, applied by the worker / API:

| Env var | Default | Where used | Meaning |
|---------|---------|------------|---------|
| `GENERATION_CONCURRENCY` | `3` | `worker/index.ts` → BullMQ `Worker({ concurrency })` | How many **story turns** one worker process runs at once |
| `GENERATION_RATE_MAX` | `10` | Same Worker `limiter` (per 60s) | Max **new** generation jobs started per minute **per worker process** |
| `CHAT_RATE_MAX` | `10` | `middleware/rate-limit.ts` action `chat` | Max chat/continue/etc. actions **per user** per 60s |
| Generation lock | 1 per player+instance | `utils/generation-lock.ts` + `play-ws.service.ts` | One in-flight turn per playthrough; second attempt → `GENERATION_IN_PROGRESS` |

Docker/systemd only **pass** these env vars into the process. They do not set container CPU by themselves.

### What happens with many users in parallel

1. Many users can stay **connected** over WebSocket (limited by box RAM/CPU/network — often dozens to low hundreds idle on a small instance).
2. When several users send turns, jobs enter the **BullMQ queue** in Redis.
3. The worker runs up to `GENERATION_CONCURRENCY` jobs at once; the rest **wait** (players stay connected; first token is delayed).
4. Same user + same instance while a turn is pending/running → rejected with `GENERATION_IN_PROGRESS` (not double-queued).
5. Same user over `CHAT_RATE_MAX` → `RATE_LIMITED`.

Users do **not** overwrite each other’s generations. They share a **global generation pool**. Extra turns queue; latency rises under load.

### Rough day-1 feel (one worker, defaults)

- ~**3** people generating at full speed at the same moment
- More can be “in flight” waiting in the queue
- Sustained start rate ≈ **~10 new turns/minute** for the whole app (not per user), per worker process
- Casual staggered chat for many users is fine; everyone mashing send at once → queueing

### Scaling ladder (honest)

| Stage | What to do |
|-------|------------|
| Early growth, still one box | Upsize EC2 (`t4g.small` → `medium` / `large`) and raise `GENERATION_CONCURRENCY` / `GENERATION_RATE_MAX` |
| One box still chokes on turns | Add **more worker processes/machines** (Spot ASG). Each worker adds its own concurrency pool. **Move Redis off localhost first.** |
| Many idle WebSocket / HTTP clients | Scale or enlarge the **API** (vertical first; ALB only after revenue) |
| Redis vs Bun RAM fight | Redis Cloud (or dedicated Redis host) |

**Do not** set `GENERATION_CONCURRENCY=20` on a tiny box — you will OOM before you get more throughput.

Rule of thumb:

- More simultaneous **turns** → concurrency + CPU/RAM, then more workers  
- More people **connected but not generating** → mostly API/WS capacity  
- LLM provider rate limits / cost will often dominate before AWS compute does  

---

## Autoscaling (later)

**Workers first (best ROI):**

- Launch template runs only `bun run start:worker`
- Spot ASG `min=0/1`, `max=N`, scale on CPU or queue depth
- BullMQ retries make Spot reclaim acceptable for generation jobs
- **Prerequisite:** shared Redis (Redis Cloud Essentials with persistence + `noeviction`, region near `ap-south-1`, TLS `rediss://`)

**API later:**

- Vertical resize first
- Second on-demand API + LB only when one box cannot hold WS+HTTP
- Prefer on-demand for API (WebSockets hate Spot reclaim)

Workers need **no** load balancer — they pull from Redis.

---

## CI/CD (GitHub Actions)

```text
push main → typecheck → deploy (SSH/SSM or ECR pull) → restart systemd → curl /health
```

**Day-1 (simplest):**

- `bun install --frozen-lockfile` + `bun run typecheck`
- SSH/SSM: pull code, `bun install --production`, restart `everlore-api` + `everlore-worker`
- Health gate must return 200

**Day-2 (cleaner):**

- Multi-stage Bun Dockerfile → ECR → pull + restart
- Worker ASG: instance refresh / new launch template version

**Secrets / prod flags:** `MONGODB_URI`, `REDIS_URL`, `JWT_SECRET`, OpenAI / OpenRouter / Pinecone, S3 keys, `CLIENT_ORIGINS`, admin creds. Set `DISABLE_OTP_RATE_LIMIT=false`, `BILLING_SIMULATION_ENABLED=false`.

---

## Code / repo work before production

Gaps relative to current `everlore-server` (no Docker/compose in-repo yet; shallow `/health`):

1. Dockerfiles + systemd units + Caddyfile + Redis conf templates
2. Deep `/health` — ping Mongo + Redis (today returns `{ ok: true }` only in `src/index.ts`)
3. Worker liveness — localhost health port or systemd `Restart=always`
4. API graceful SIGTERM (close WS / Mongo / Redis cleanly)
5. Production `CLIENT_ORIGINS`
6. Atlas IP allowlist + strong `JWT_SECRET`
7. Redis `noeviction` + AOF + memory cap
8. Tune `GENERATION_CONCURRENCY` for instance size

No app rewrite for multi-API later: Redis pub/sub already fans out WS events (`src/services/play-ws.service.ts`).

---

## Cost control checklist

- Same region for EC2 + S3 (`ap-south-1`) + Atlas nearby
- No NAT, ALB, Route 53, ElastiCache, Fargate on day-1
- On-box Redis until multi-host workers
- ARM small on-demand for API; Spot only for workers later
- ECR lifecycle: keep last N images
- Minimal CloudWatch (journald is enough at first)
- Expect LLM spend to dominate AWS compute

---

## Implementation order

1. Dockerfile(s), Caddyfile, systemd units, Redis conf
2. Harden `/health` (+ worker liveness)
3. Provision EC2 + EIP + SG + Porkbun `A` + Atlas allowlist
4. Manual smoke: WebSocket turn end-to-end through local Redis
5. GitHub Actions deploy + secrets
6. Later: Redis Cloud + worker Spot ASG when one box is not enough

---

## Related docs

- [INFRASTRUCTURE.md](./INFRASTRUCTURE.md) — broader topology, resource tables, compose sketches
- [WORKERS.md](./WORKERS.md) — queues and processors
- [CONFIGURATION.md](./CONFIGURATION.md) — full env reference
- [SECURITY.md](./SECURITY.md) — auth and rate limits
