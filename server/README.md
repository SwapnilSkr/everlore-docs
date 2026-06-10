# Everlore Server Documentation

Backend docs for the Bun/Elysia API + BullMQ workers.

**Start here for the memory system:** [../system-guide/README.md](../system-guide/README.md)  
**File-by-file code reference:** [../code-reference/SERVER.md](../code-reference/SERVER.md)

---

## Core architecture

| Document | Description |
|----------|-------------|
| [OVERVIEW.md](./OVERVIEW.md) | Stack, component diagram, design principles |
| [DATA_MODEL.md](./DATA_MODEL.md) | Mongo collections, entities, events, memories, summaries |
| [SERVICES.md](./SERVICES.md) | All services + providers (current list) |
| [WORKERS.md](./WORKERS.md) | Queues, processors, cron jobs, turn pipeline |
| [MEMORY_ARCHITECTURE.md](./MEMORY_ARCHITECTURE.md) | Codex + four prompt layers (foundation doc; see system-guide for full picture) |

## API & integration

| Document | Description |
|----------|-------------|
| [SCHEMAS.md](./SCHEMAS.md) | Canonical request/response JSON shapes |
| [API.md](./API.md) | HTTP + WebSocket protocol |
| [SECURITY.md](./SECURITY.md) | Auth, JWT, rate limits |
| [CONFIGURATION.md](./CONFIGURATION.md) | Environment variables |

## AI models

| Document | Description |
|----------|-------------|
| [SFW_MODELS.md](./SFW_MODELS.md) | SFW narration models + testing |
| [NSFW_MODELS.md](./NSFW_MODELS.md) | NSFW routing + models |
| [TTS_MODELS.md](./TTS_MODELS.md) | Text-to-speech |
| [IMAGE_MODELS.md](./IMAGE_MODELS.md) | Cover/avatar generation |

## Operations

| Document | Description |
|----------|-------------|
| [INFRASTRUCTURE.md](./INFRASTRUCTURE.md) | Deployment, scaling |

---

## Quick start for developers

1. **How does a turn work?** → [../system-guide/02-one-turn-journey.md](../system-guide/02-one-turn-journey.md)
2. **What changed recently?** → [../memory/CHECKLIST.md](../memory/CHECKLIST.md) + [../memory/HANDOFF.md](../memory/HANDOFF.md)
3. **Adding an endpoint?** → [SERVICES.md](./SERVICES.md) + [API.md](./API.md)
4. **Debugging memory?** → `GET /admin/events/:eventId/projections`, `scripts/rewind-audit.ts`

---

## System diagram

```text
Flutter app ──HTTP/WS──► API (src/index.ts)
                              │
                    enqueue   │
                              ▼
                         Workers
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
           MongoDB         Redis          Pinecone
```

---

## When updating code, update docs

| Change | Doc |
|--------|-----|
| New route or response shape | [API.md](./API.md) + [SCHEMAS.md](./SCHEMAS.md) |
| New collection/field | [DATA_MODEL.md](./DATA_MODEL.md) |
| New service | [SERVICES.md](./SERVICES.md) |
| Worker/queue change | [WORKERS.md](./WORKERS.md) |
| Env var | [CONFIGURATION.md](./CONFIGURATION.md) |
| Memory behavior | [../memory/CHECKLIST.md](../memory/CHECKLIST.md) + [../system-guide/](../system-guide/) |
