# Everlore Server - Architecture Overview

## What is Everlore?

Everlore is an AI-powered narrative roleplay platform. It allows players to engage with AI-driven worlds and characters through immersive, persistent story experiences. The system supports both SFW and NSFW content with appropriate content classification and model routing.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
│                    (Flutter App, Web Client, etc.)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ HTTP / WebSocket
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API SERVER (Elysia)                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Auth Routes  │  │ Template     │  │ Instance     │  │ Chronicle Routes │ │
│  │ /auth/*      │  │ Routes       │  │ Routes       │  │ /chronicle/*     │ │
│  └──────────────┘  │ /templates/* │  │ /instances/* │  └──────────────────┘ │
│  ┌──────────────┐  └──────────────┘  └──────────────┘  ┌──────────────────┐ │
│  │ WS Routes    │                                      │ WS /ws/play      │ │
│  └──────────────┘                                      └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
           ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
           │   MongoDB   │    │    Redis    │    │  Pinecone   │
           │  (Primary)  │    │  (Cache/   │    │   (Vector   │
           │             │    │   Queue)   │    │    Index)   │
           └─────────────┘    └─────────────┘    └─────────────┘
                                        │
                                        ▼
                           ┌─────────────────────────┐
                           │    BULLMQ QUEUES        │
                           │  ┌─────────────────┐    │
                           │  │ generation      │    │
                           │  │ memory-curation │    │
                           │  │ scene-summary   │    │
                           │  │ maintenance     │    │
                           │  └─────────────────┘    │
                           └─────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          WORKER CLUSTER                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │ Generation      │  │ Memory          │  │ Summary         │             │
│  │ Processor       │  │ Processor       │  │ Processor       │             │
│  │ (Concurrency: 3)│  │ (Concurrency: 5)│  │ (Concurrency: 2)│             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│  ┌─────────────────┐                                                        │
│  │ Maintenance     │                                                        │
│  │ Processor       │                                                        │
│  │ (Concurrency: 1)│                                                        │
│  └─────────────────┘                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                           ┌─────────────────────────┐
                           │    LLM PROVIDERS        │
                           │  ┌─────────────────┐    │
                           │  │ OpenAI          │    │
                           │  │ (GPT-4o, etc.)  │    │
                           │  └─────────────────┘    │
                           │  ┌─────────────────┐    │
                           │  │ OpenRouter      │    │
                           │  │ (Mythalion,     │    │
                           │  │  etc.)          │    │
                           │  └─────────────────┘    │
                           └─────────────────────────┘
```

## Core Components

### 1. API Server (`src/index.ts`)
- **Framework**: Elysia (Bun-based, high-performance)
- **Purpose**: HTTP API + WebSocket real-time communication
- **Key Features**:
  - JWT authentication
  - CORS support for multi-origin clients
  - Error handling with appropriate HTTP status codes
  - Health check endpoint

### 2. Database Layer

#### MongoDB (Primary Data Store)
- **Purpose**: Persistent storage for all application data
- **Collections**:
  - `users` - User accounts and preferences
  - `world_templates` - World templates/blueprints
  - `world_instances` - Player's active game instances
  - `events` - Game event history
  - `memories` - Extracted long-term memories
  - `scene_summaries` - Compressed scene narratives
  - `dead_letter_jobs` - Failed job tracking

#### Redis (Cache & Message Broker)
- **Purpose**: Session caching, rate limiting, pub/sub, job queues
- **Key Uses**:
  - Session state caching (1-hour TTL)
  - Generation locks (prevents duplicate requests)
  - Rate limiting counters
  - WebSocket pub/sub for real-time events
  - BullMQ job queue backend

#### Pinecone (Vector Database)
- **Purpose**: Semantic search for lore and memories
- **Namespaces**:
  - `lore_{templateId}` - Template-specific world lore
  - `mem_{instanceId}` - Instance-specific memories
- **Embedding Model**: OpenAI text-embedding-3-small

### 3. Queue System (BullMQ)

Four distinct queues for background processing:

| Queue | Purpose | Concurrency | Priority |
|-------|---------|-------------|----------|
| `generation` | AI narrative generation | 3 | 1 (high) |
| `memory-curation` | Extract & store memories | 5 | 5 |
| `scene-summary` | Compress long scenes | 2 | 10 |
| `maintenance` | Scheduled cleanup tasks | 1 | 20 |

### 4. Worker Cluster (`worker/`)

Separate process that consumes from BullMQ queues:
- **Generation Worker**: Handles AI chat responses
- **Memory Worker**: Extracts important facts from conversations
- **Summary Worker**: Compresses scene history
- **Maintenance Worker**: Scheduled cleanup and optimization

## Request Flow

### Chat Request Flow

```
1. Client → WS /ws/play (chat action)
         │
2. API Server validates token, checks rate limit
         │
3. API Server sets Redis lock
         │
4. API Server enqueues job to BullMQ (generation queue)
         │
5. API Server returns jobId to client
         │
6. Worker picks up job from queue
         │
7. Worker queries Pinecone (RAG) for relevant lore/memories
         │
8. Worker builds prompt with context
         │
9. Worker calls LLM (OpenAI/OpenRouter)
         │
10. Worker parses structured response
         │
11. Worker persists event to MongoDB
         │
12. Worker updates world state in MongoDB
         │
13. Worker updates Redis session cache
         │
14. Worker releases Redis lock
         │
15. Worker publishes event to Redis pub/sub
         │
16. API Server (via subscriber) broadcasts to client's WS
         │
17. Client receives generation_complete event
```

### Memory Curation Flow (Async)

```
After generation completes:

1. Worker enqueues memory-curation job (delayed 1s)
         │
2. Memory Worker picks up job
         │
3. Worker calls LLM to extract important facts
         │
4. Worker embeds extracted memories
         │
5. Worker upserts vectors to Pinecone
         │
6. Worker inserts memory documents to MongoDB
         │
7. Worker publishes memories_curated event to client
```

## Key Design Decisions

### 1. Separation of API and Workers
- API handles HTTP/WS I/O quickly
- Workers handle CPU-intensive AI generation
- Allows independent scaling

### 2. Session Caching Strategy
- Redis caches deserialized session data
- 1-hour TTL with automatic refresh
- Reduces MongoDB load for active instances

### 3. RAG (Retrieval Augmented Generation)
- Vector similarity search for context retrieval
- Separate namespaces per template/instance
- Access counts tracked for importance scoring

### 4. Structured Output
- LLM responses use JSON Schema validation
- Automatic repair for malformed responses
- State mutations validated and applied atomically

### 5. Content Classification
- Rule-based NSFW detection (keywords + patterns)
- Scene momentum tracking (recent intimate scenes)
- Model routing based on content classification

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Runtime | Bun | JavaScript/TypeScript runtime |
| API Framework | Elysia | HTTP/WebSocket server |
| Auth | @elysiajs/jwt | JWT token handling |
| Database | MongoDB | Primary data persistence |
| Cache | Redis (ioredis) | Session cache, pub/sub |
| Queue | BullMQ | Background job processing |
| Vector DB | Pinecone | Semantic search |
| LLM | OpenAI / OpenRouter | AI generation |
| Password Hash | Argon2 | Secure password storage |
| Validation | Elysia TypeBox | Runtime type checking |

## Scaling Considerations

### Horizontal Scaling
- **API Servers**: Can run multiple instances behind load balancer
- **Workers**: Can scale worker processes independently
- **Redis**: Use Redis Cluster for queue scaling
- **MongoDB**: Use replica sets for read scaling

### Bottlenecks
- LLM API rate limits
- Vector DB query latency
- Memory curation volume

### Mitigation Strategies
- Limiter config on workers
- Connection pooling
- Batch embedding operations
- Prioritized queues
