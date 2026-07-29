# Backend & Workers

Server architecture, data stores, queues, and key services.  
Authoritative deep docs: [API.md](../server/API.md) · [SERVICES.md](../server/SERVICES.md) · [WORKERS.md](../server/WORKERS.md) · [BILLING.md](../server/BILLING.md) · [DATA_MODEL.md](../server/DATA_MODEL.md).

---

## Stack

| Piece | Technology |
|-------|------------|
| API + WebSocket | Bun + Elysia |
| Primary database | MongoDB |
| Vector search | Pinecone |
| Job queues | BullMQ + Redis |
| AI calls | OpenAI / OpenRouter (centralized in `src/ai/`) |
| Billing | Story Ink ledger + Google Play verify / RTDN |
| File storage | S3 + CloudFront (avatars, covers) |

---

## Architecture diagram

```text
┌─────────────┐     WebSocket/REST      ┌──────────────────┐
│ Flutter app │ ◄──────────────────────►│  everlore-server │
└─────────────┘                         │  (API process)   │
                                        │  + Ink reserve   │
                                        └────────┬─────────┘
                                                 │ enqueue
                                                 ▼
                                        ┌──────────────────┐
                                        │  worker process  │
                                        │  BullMQ consumers│
                                        │  settle / release│
                                        └────────┬─────────┘
                    ┌────────────────────────────┼────────────────────────────┐
                    ▼                            ▼                            ▼
              ┌──────────┐               ┌──────────────┐              ┌───────────┐
              │  MongoDB │               │   Pinecone   │              │   Redis   │
              └──────────┘               └──────────────┘              └───────────┘
```

Two processes typically run: **API** (handles HTTP/WS + Ink reserve) and **worker** (AI jobs + settle/release).

---

## MongoDB collections (memory-related + billing)

| Collection | Role |
|------------|------|
| `events` | Every turn — source of truth |
| `memories` | Long-term fact atoms (+ `updates_memory_ids` Slice 1) |
| `characters` | Codex cards |
| `entities` | Graph nodes (people, places, …) |
| `entity_edges` | Graph relationships incl. **kinship** |
| `relation_candidates` | Narrator review queue (not canon) |
| `signal_ledger` | Per-turn FP/FN detector metrics |
| `scene_summaries` / `chapter_summaries` / `arc_summaries` | Summary tiers |
| `story_calendars` / `timeline_branches` | Time |
| `world_instances` / `world_templates` | Sessions + blueprints |
| `ink_ledger` / `billing_entitlements` / `store_purchases` | Story Ink |

Indexes defined in `config/mongo-indexes.ts`.

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
| `generation` | 3 | `generation.processor.ts` | Main turns, replay, side chat, world_action |
| `memory-curation` | 5 | `memory.processor.ts` | Extract + embed memories |
| `scene-summary` | 2 | `summary.processor.ts` | Scene, chapter, arc summaries |
| `maintenance` | 1 | `maintenance.processor.ts` | Decay, dedup, repair, audits, checkpoints |

### Scheduled maintenance (cron)

| Job | Schedule | What it does |
|-----|----------|--------------|
| Continuity audit scheduler | Daily 02:30 | Fan out drift checks across active worlds |
| Importance decay | Daily 03:00 | Archive stale low-importance memories |
| Dedup scheduler | Weekly Sun 04:00 | Merge near-duplicate memories per instance |
| Summary repair | Every 15 min | Fix stuck summary jobs |
| Projection checkpoint scheduler | Hourly :15 | Fan out projection checkpoints |

---

## Key services (by responsibility)

### Turn orchestration & billing

| Service | File | Role |
|---------|------|------|
| `generationService` | `generation.service.ts` | Thin dispatch, load play feed |
| `playWsService` | `play-ws.service.ts` | WS protocol, locks, **reserve Ink** |
| `billingService` | `billing.service.ts` | Wallet, catalog, reserve/settle/release, Play verify |
| `buildContextPacket` | `context-packet.service.ts` | Assemble briefing before prompt |
| `buildPrompt` | `prompt-builder.ts` | Token-budgeted LLM messages |
| Generation worker | `generation.processor.ts` | **Prose stream**, post-prose pipeline, **settle** |

### Memory & chronicle

| Service | File | Role |
|---------|------|------|
| `memoryService` | `memory.service.ts` | Rewind, edit, recap, threads, events API |
| Memory worker | `memory.processor.ts` | Extract atoms; materialize version links |
| `queryRag` | `rag.provider.ts` | Hybrid retrieval |
| `memorySupersession` | `memory-supersession.service.ts` | Retire vectors + mark superseded ids |

### Characters, kinship & graph

| Service | File | Role |
|---------|------|------|
| `characterCodexService` | `character-codex.service.ts` | Deltas, ranking, pinning, compaction |
| `entityGraphService` | `entity-graph.service.ts` | Entities, edges, location facts, rewind repair |
| `kinshipGraphService` | `kinship-graph.service.ts` | Family graph, premise seed, relatives brief |
| `relationCandidateService` | `relation-candidate.service.ts` | Open review queue |
| Codex / kinship extractors | `worker/lib/*` | Post-prose LLM + deterministic detectors |

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
| `continuityAuditService` | `continuity-audit.service.ts` | Cross-checks, admin report |
| `adminService` | `admin.service.ts` | Projections inspect, drift status, Ink grants |
| Maintenance worker | `maintenance.processor.ts` | Decay, dedup, graph/link repair, checkpoints |

---

## REST API surfaces (high level)

| Area | Router | Highlights |
|------|--------|------------|
| Chronicle | `chronicle.routes.ts` | Events, memories, recap, bonds, places, calendar, side chats, edit/rewind/track, **`/kinship`**, **`/relation-candidates`** |
| Billing | `billing.routes.ts` | `/catalog`, `/me`, `/google/verify`, `/simulate-purchase`, RTDN |
| Instances / templates / personas / auth | respective routes | See API.md |
| Admin | `admin.routes.ts` | Basic auth; continuity audits; projections |

---

## WebSocket play protocol

**File:** `play-ws.service.ts`

| Action | Billable |
|--------|----------|
| `chat`, `continue`, `world_action`, `side_chat`, `replay` | Story turn (Ink reserve) |
| `load_instance`, `ping` | No |

Player actions enqueue worker jobs; streaming frames push back on the same socket. Session cached in Redis — busted on rewind/reset/edit that changes canon.

---

## Generation post-prose pipeline (worker)

Narration is a **prose stream** (tokens via Redis → WS). After stream end, the worker does **not** treat the turn as a structured JSON response body. Rough order:

1. Prose hygiene / stream filters / choice-tail parse  
2. Scene metadata extractor (presence, location, time, choices)  
3. Deterministic signals (movement, time-skip, party, kinship, presence gaps)  
4. **Entity adjudication** for strong person-candidate terms  
5. Persist event + fold presence / cursors  
6. Fire-and-forget: graph sync, kinship apply, relation candidates, anomalies, **signal_ledger**  
7. Enqueue memory-curation (+ maybe scene summary)  
8. **Billing settle** (or release on final pre-stream failure)

Full map: [WORKERS.md](../server/WORKERS.md).

---

## Best practices used

| Practice | Where |
|----------|-------|
| **Event sourcing for rewind** | Codex / kinship deltas ledgered on events |
| **Projection rebuild** | Summaries, memories, graph repair on mutation |
| **Fail-closed privacy** | Side-chat memory gate in RAG |
| **Bounded hot paths** | Context packet caps + token floors |
| **Async heavy work** | Memory, summaries, compaction, signal ledger off TTFT path |
| **Ink reservations** | Reserve on WS; settle/release in worker |
| **Hybrid retrieval** | Vector + keyword + entity + place + RRF |
| **Centralized AI client** | Model map, embeddings, streaming in `src/ai/` |

---

## Verification harnesses

| Tool | Purpose |
|------|---------|
| `scripts/rewind-audit.ts` | Clone instance, rewind, assert invariants |
| `bun run audit:choices` / `audit:location` / `audit:codex-dedup` / `audit:replay-edit` | Live / integration audits |
| `bun run audit:memory-links` | Version-link lifecycle |
| `bun run audit:side-chat-privacy` / `audit:carding-routing` | Privacy + carding guards |
| `continuityAuditService` | Cross-projection consistency report |
| `bun run typecheck` / `flutter analyze lib` | Static checks |
