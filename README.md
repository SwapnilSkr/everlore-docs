# Everlore Docs

Documentation for the Everlore Flutter client and Bun/Elysia server.

---

## Start here

| Audience | Read |
|----------|------|
| **Understand the whole system** | [system-guide/README.md](./system-guide/README.md) |
| **Find a specific source file** | [code-reference/README.md](./code-reference/README.md) |
| **Continue development / agents** | [memory/HANDOFF.md](./memory/HANDOFF.md) + [memory/CHECKLIST.md](./memory/CHECKLIST.md) |

---

## Documentation map

```text
everlore-docs/
├── system-guide/       ← Product + memory system (plain language)
├── code-reference/     ← File-by-file server + client map
├── memory/             ← Infinite memory roadmap, checklist, handoff
├── server/             ← API, data model, workers, config, models
├── client/             ← Flutter architecture + features
└── visual-guide/       ← Pixel-level UI notes (may lag — check system-guide first)
```

---

## Server (`everlore-server/`)

| Doc | Topic |
|-----|-------|
| [server/README.md](./server/README.md) | Index |
| [server/OVERVIEW.md](./server/OVERVIEW.md) | Architecture, routes, collections, narration pipeline |
| [server/SERVICES.md](./server/SERVICES.md) | All services (current) |
| [server/DATA_MODEL.md](./server/DATA_MODEL.md) | Mongo collections |
| [server/WORLD_MODEL.md](./server/WORLD_MODEL.md) | Witness / entity / canon model |
| [server/API.md](./server/API.md) | HTTP + WebSocket |
| [server/SCHEMAS.md](./server/SCHEMAS.md) | Request/response JSON shapes (canonical) |
| [server/WORKERS.md](./server/WORKERS.md) | Queues & processors |
| [server/BILLING.md](./server/BILLING.md) | Story Ink + Google Play (canonical) |
| [server/DEPLOYMENT.md](./server/DEPLOYMENT.md) | Cheap AWS prod deploy + scaling |
| [server/CONFIGURATION.md](./server/CONFIGURATION.md) | Environment variables |
| [server/NSFW_MODELS.md](./server/NSFW_MODELS.md) | NSFW routing + intent deferral |

## Client (`everlore/`)

| Doc | Topic |
|-----|-------|
| [client/README.md](./client/README.md) | Index |
| [client/features/play-feature.md](./client/features/play-feature.md) | Play screen |
| [client/features/billing-feature.md](./client/features/billing-feature.md) | Membership / Story Ink UI |
| [client/features/chronicle-feature.md](./client/features/chronicle-feature.md) | Lore Tome (7 tabs) |
| [client/architecture/routing.md](./client/architecture/routing.md) | Routes |

## Memory program

| Doc | Topic |
|-----|-------|
| [memory/CHECKLIST.md](./memory/CHECKLIST.md) | Phases 1–10 status |
| [memory/HANDOFF.md](./memory/HANDOFF.md) | Agent handoff |
| [memory/MEMORY_ARCHITECTURE.md](./memory/MEMORY_ARCHITECTURE.md) | Target vision |
| [memory/KINSHIP_GRAPH.md](./memory/KINSHIP_GRAPH.md) | Typed kinship / entity edges |

## Playtest

| Doc | Topic |
|-----|-------|
| [playtest/PLAYTEST_FINDINGS_MATRIX_abcd.md](./playtest/PLAYTEST_FINDINGS_MATRIX_abcd.md) | Regression matrix (runs a–d) |

---

## Repos

Three separate git repos under `rpg-ai/`:

- `everlore/` — Flutter app  
- `everlore-server/` — API + workers  
- `everlore-docs/` — this repo  

---

## Auth (quick reference)

JWT from `/auth/login`, `/auth/register`, `/auth/google`, or `/auth/otp/verify`.  
Stored in secure storage; used for REST + `WS /ws/play?token=...`.

Details: [server/SECURITY.md](./server/SECURITY.md), [client/core/core-layer.md](./client/core/core-layer.md)
