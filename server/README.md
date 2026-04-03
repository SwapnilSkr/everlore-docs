# Everlore Documentation

Complete architecture documentation for the Everlore AI narrative platform — covering both the server backend and the Flutter frontend client.

---

## Server Documentation

### Core Architecture

| Document | Description |
|----------|-------------|
| [OVERVIEW.md](OVERVIEW.md) | High-level system architecture, component overview, and design decisions |
| [DATA_MODEL.md](DATA_MODEL.md) | Database schemas, collections, indexes, and data flow |
| [SERVICES.md](SERVICES.md) | Service layer architecture and business logic |
| [WORKERS.md](WORKERS.md) | Background job processors and queue system |

### API & Integration

| Document | Description |
|----------|-------------|
| [API.md](API.md) | HTTP endpoints, WebSocket protocol, and request/response formats |

### Operations

| Document | Description |
|----------|-------------|
| [INFRASTRUCTURE.md](INFRASTRUCTURE.md) | Deployment options, scaling, and infrastructure setup |
| [CONFIGURATION.md](CONFIGURATION.md) | Environment variables, feature flags, and configuration |
| [SECURITY.md](SECURITY.md) | Security considerations, authentication, and best practices |

---

## Frontend Documentation (Flutter Client)

### Architecture

| Document | Description |
|----------|-------------|
| [architecture/overview.md](architecture/overview.md) | Tech stack, directory structure, data flow, design decisions, API contract |
| [architecture/state-management.md](architecture/state-management.md) | Cubit pattern, all cubits, optimistic updates, adding new cubits |
| [architecture/routing.md](architecture/routing.md) | Route table, navigation API, navigation flows |
| [architecture/theme.md](architecture/theme.md) | Color palette, component themes, hardcoded colors |

### Core Layer

| Document | Description |
|----------|-------------|
| [core/core-layer.md](core/core-layer.md) | Config, Auth, API Client, WebSocket Manager, Secure Storage, Local DB |

### Features

| Document | Description |
|----------|-------------|
| [features/home-feature.md](features/home-feature.md) | World instance dashboard, WorldCard, HomeCubit |
| [features/play-feature.md](features/play-feature.md) | Real-time gameplay chat, NarrativeBubble, PlayerInput, WorldStateBar, PlayCubit |
| [features/chronicle-feature.md](features/chronicle-feature.md) | Timeline & memory browser, MemoryCard, EditDialog, ChronicleCubit |
| [features/templates-feature.md](features/templates-feature.md) | Template browsing, template details, instance creation |

### Shared

| Document | Description |
|----------|-------------|
| [shared/models.md](shared/models.md) | User, GameEvent, Memory, WorldInstance, WorldTemplate, StatDefinition |
| [shared/widgets.md](shared/widgets.md) | ErrorBanner, StatBar, LoadingShimmer |

### Guides

| Document | Description |
|----------|-------------|
| [guides/getting-started.md](guides/getting-started.md) | Setup, conventions, common tasks, testing, known issues |

## Quick Start

1. **New Developer?** Start with [OVERVIEW.md](OVERVIEW.md)
2. **Building Features?** Check [SERVICES.md](SERVICES.md) and [API.md](API.md)
3. **Deploying?** Read [INFRASTRUCTURE.md](INFRASTRUCTURE.md) and [CONFIGURATION.md](CONFIGURATION.md)
4. **Security Review?** See [SECURITY.md](SECURITY.md)

## System Overview

```
┌─────────────────┐
│   Client Apps   │
└────────┬────────┘
         │ HTTP / WebSocket
         ▼
┌─────────────────┐
│   API Server    │  ← Elysia (Bun)
│  (src/index.ts) │
└────────┬────────┘
         │
    ┌────┴────┬──────────┐
    ▼         ▼          ▼
┌────────┐ ┌────────┐ ┌──────────┐
│ MongoDB│ │ Redis  │ │ Pinecone │
└────────┘ └────────┘ └──────────┘
                │
                ▼
         ┌────────────┐
         │  BullMQ    │
         │   Queues   │
         └────────────┘
                │
                ▼
┌─────────────────────────┐
│      Worker Cluster     │  ← Background processing
│    (worker/index.ts)    │
└─────────────────────────┘
```

## Technology Stack

| Layer | Technology |
|-------|------------|
| Runtime | Bun |
| API Framework | Elysia |
| Database | MongoDB |
| Cache | Redis |
| Queue | BullMQ |
| Vector DB | Pinecone |
| LLM | OpenAI / OpenRouter |
| Auth | JWT (Argon2 for passwords) |

## Architecture Principles

1. **Separation of Concerns**: API handles I/O, Workers handle processing
2. **Stateless API**: Session data in Redis, enables horizontal scaling
3. **Event-Driven**: WebSocket pub/sub for real-time updates
4. **RAG**: Vector search for context-aware AI responses
5. **Structured Output**: JSON Schema validation for LLM responses

## Directory Structure

```
everlore-server/
├── src/
│   ├── config/          # Database connections, env
│   ├── middleware/      # Auth, rate limiting
│   ├── queues/          # BullMQ queue definitions
│   ├── routes/          # HTTP/WebSocket handlers
│   ├── schemas/         # TypeBox validation schemas
│   ├── services/        # Business logic
│   └── utils/           # Helper functions
├── worker/
│   ├── lib/             # Worker utilities
│   ├── processors/      # Job processors
│   └── index.ts         # Worker entry point
└── everlore-docs/       # This documentation
```

## Contributing

When updating the codebase, please keep this documentation in sync:

- API changes → Update [API.md](API.md)
- Database changes → Update [DATA_MODEL.md](DATA_MODEL.md)
- New services → Update [SERVICES.md](SERVICES.md)
- Worker changes → Update [WORKERS.md](WORKERS.md)
- Config changes → Update [CONFIGURATION.md](CONFIGURATION.md)
- Security changes → Update [SECURITY.md](SECURITY.md)

## License

[Your License Here]

## Support

For questions or issues:
- Development: Check relevant documentation
- Security: See [SECURITY.md](SECURITY.md) reporting guidelines
- Operations: Review [INFRASTRUCTURE.md](INFRASTRUCTURE.md)
