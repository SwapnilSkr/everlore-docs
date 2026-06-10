# Backend & Workers

Server architecture, data stores, queues, and key services.

---

## Stack

| Piece | Technology |
|-------|------------|
| API + WebSocket | Bun + Elysia |
| Primary database | MongoDB |
| Vector search | Pinecone |
| Job queues | BullMQ + Redis |
| AI calls | OpenAI / OpenRouter (centralized in `src/ai/`) |
| File storage | S3 + CloudFront (avatars, covers) |

---

## Architecture diagram

```text
┌─────────────┐     WebSocket/REST      ┌──────────────────┐
│ Flutter app │ ◄──────────────────────►│  everlore-server │
└─────────────┘                         │  (API process)   │
                                        └────────┬─────────┘
                                                 │ enqueue
                                                 ▼
                                        ┌──────────────────┐
                                        │  worker process  │
                                        │  BullMQ consumers│
                                        └────────┬─────────┘
                    ┌────────────────────────────┼────────────────────────────┐
                    ▼                            ▼                            ▼
              ┌──────────┐               ┌──────────────┐              ┌───────────┐
              │  MongoDB │               │   Pinecone   │              │   Redis   │
              └──────────┘               └──────────────┘              └───────────┘
```

Two processes typically run: **API** (handles HTTP/WS) and **worker** (handles AI jobs).

---

## MongoDB collections (memory-related)

| Collection | Role |
|------------|------|
| `events` | Every turn — source of truth |
| `memories` | Long-term fact atoms |
| `characters` | Codex cards |
| `entities` | Graph nodes (people, places, …) |
| `entity_edges` | Graph relationships |
| `scene_summaries` | 12-turn blocks |
| `chapter_summaries` | 8-scene rollups |
| `arc_summaries` | 4-chapter rollups |
| `story_calendars` | In-world calendar defs |
| `timeline_branches` | Alternate realities |
| `world_instances` | Play session state + cursors |
| `world_templates` | World blueprint + global lore |

Indexes defined in `config/mongo-indexes.ts` (text search on memories, timeline fields, etc.).

---

## Pinecone namespaces

| Namespace | Contents |
|-----------|----------|
| `lore_{templateId}` | Static world lore (shared across instances) |
| `mem_{instanceId}` | Memory embeddings per playthrough |
| `sum_{instanceId}` | Scene/chapter/arc summary embeddings |

---

## Job queues

**Defined in:** `src/queues/index.ts`  
**Workers in:** `worker/index.ts`

| Queue | Concurrency | Processor | Jobs |
|-------|-------------|-----------|------|
| `generation` | 3 | `generation.processor.ts` | Main turns, replay, side chat |
| `memory-curation` | 5 | `memory.processor.ts` | Extract + embed memories |
| `scene-summary` | 2 | `summary.processor.ts` | Scene, chapter, arc summaries |
| `maintenance` | 1 | `maintenance.processor.ts` | Decay, dedup, repair, audits |

### Scheduled maintenance (cron)

| Job | Schedule | What it does |
|-----|----------|--------------|
| Continuity audit scheduler | Daily 02:30 | Fan out drift checks across active worlds |
| Importance decay | Daily 03:00 | Archive stale low-importance memories |
| Dedup scheduler | Weekly Sun 04:00 | Merge near-duplicate memories per instance |
| Summary repair | Every 15 min | Fix stuck summary jobs |

---

## Key services (by responsibility)

### Turn orchestration

| Service | File | Role |
|---------|------|------|
| `generationService` | `generation.service.ts` | Thin dispatch, load play feed |
| `buildContextPacket` | `context-packet.service.ts` | Assemble briefing before prompt |
| `buildPrompt` | `prompt-builder.ts` | Token-budgeted LLM messages |
| Generation worker | `generation.processor.ts` | Stream AI, save event, enqueue follow-ups |

### Memory & chronicle

| Service | File | Role |
|---------|------|------|
| `memoryService` | `memory.service.ts` | Rewind, edit, recap, threads, events API |
| Memory worker | `memory.processor.ts` | Extract atoms post-turn |
| `queryRag` | `rag.provider.ts` | Hybrid retrieval |
| `memorySupersession` | `memory-supersession.service.ts` | Retire stale vectors when codex contradicts |

### Characters & graph

| Service | File | Role |
|---------|------|------|
| `characterCodexService` | `character-codex.service.ts` | Deltas, ranking, pinning, compaction |
| `entityGraphService` | `entity-graph.service.ts` | Entities, edges, location facts, rewind repair |
| Codex extractor | `worker/lib/character-codex-extractor.ts` | LLM → deltas |
| Codex compactor | `worker/lib/codex-compactor.ts` | Shrink long fact lists |

### Time, place, side chat

| Service | File | Role |
|---------|------|------|
| `timeService` | `time.service.ts` | Calendar, timelines, anchors |
| `locationService` | `location.service.ts` | Place journal APIs |
| `sideChatService` | `side-chat.service.ts` | Thread list/history |
| Side chat worker | `side-chat.processor.ts` | Private in-character replies |

### Quality & ops

| Service | File | Role |
|---------|------|------|
| `continuityAuditService` | `continuity-audit.service.ts` | 8 cross-checks, admin report |
| `adminService` | `admin.service.ts` | Projections inspect, drift status |
| Maintenance worker | `maintenance.processor.ts` | Decay, dedup, graph repair |

---

## REST API surfaces (Chronicle)

**Router:** `routes/chronicle.routes.ts`

| Endpoint | Purpose |
|----------|---------|
| `GET /chronicle/events/:id` | Timeline |
| `GET /chronicle/memories/:id` | Echoes (+ search params) |
| `GET /chronicle/recap/:id` | Story so far |
| `GET /chronicle/threads/:id` | Open/resolved threads |
| `GET /chronicle/relationships/:id` | Bonds ledger |
| `GET /chronicle/relationships/:id/:charId/memories` | Character memory view |
| `GET /chronicle/locations/:id` | Places index |
| `GET /chronicle/locations/:id/:locId` | Place journal |
| `GET /chronicle/calendar/:id` | Almanac data |
| `PUT /chronicle/calendar/:id/timeline/active` | Switch reality |
| `GET /chronicle/side-chats/:id[/:charId]` | Side chat threads |
| `POST /chronicle/rewind/:id` | Roll back |
| `PUT /chronicle/event/:id` | Edit turn |
| `PUT /chronicle/memory/:id` | Edit memory |
| `POST /chronicle/replay/select/:id` | Commit replay variant |

### Admin (Basic auth)

| Endpoint | Purpose |
|----------|---------|
| `GET /admin/events/:eventId/projections` | Inspect all projections from one event |
| `GET /admin/instances/:id/continuity-audit` | Run consistency check |
| `GET /admin/instances/continuity-audits` | List worlds by drift status |

---

## WebSocket play protocol

**File:** `play-ws.service.ts`

Player actions enqueue worker jobs; streaming frames push back on the same socket.

Session cached in Redis — busted on rewind/reset/edit that changes canon.

---

## Best practices used

| Practice | Where |
|----------|-------|
| **Event sourcing for rewind** | Codex deltas ledgered on events |
| **Projection rebuild** | Summaries, memories, graph repair on mutation |
| **Fail-closed privacy** | Side-chat memory gate in RAG |
| **Bounded hot paths** | Context packet caps + token floors |
| **Async heavy work** | Memory curation, summaries, compaction off turn path |
| **Hybrid retrieval** | Vector + keyword + entity + place + RRF fusion |
| **Deterministic summary IDs** | Rebuild overwrites same Pinecone vector |
| **Read-only drift detection** | Daily audit logs warnings, no auto-mutate |
| **Rules-first hygiene** | LLM repair only on failure |
| **Centralized AI client** | Model map, embeddings, streaming in `src/ai/` |

---

## Verification harnesses

| Tool | Purpose |
|------|---------|
| `scripts/rewind-audit.ts` | Clone instance, rewind, assert 27+ invariants |
| `continuityAuditService` | Cross-projection consistency report |
| `bun run typecheck` / `flutter analyze lib` | Static checks |
