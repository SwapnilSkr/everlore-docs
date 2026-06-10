# Everlore Server — Service Layer

How the API is organized: **routes → controllers → services → workers**.

For the full memory system in plain language, see [../system-guide/](../system-guide/README.md).  
For file-by-file notes and optimizations, see [../code-reference/SERVER.md](../code-reference/SERVER.md).

---

## Request flow

```text
HTTP / WebSocket
    → routes/*.routes.ts     (validation, auth plugins)
    → controllers/*.controller.ts   (orchestration)
    → services/*.service.ts  (domain logic, Mongo/Redis/Pinecone)
    → queues (async)         → worker/processors/*.ts
```

WebSocket play uses `play-ws.controller.ts` → `play-ws.service.ts` → `generation.service.ts` (enqueue only).

---

## Controllers

| File | Routes | Responsibility |
|------|--------|----------------|
| `auth.controller.ts` | `/auth/*` | Register, login, Google, OTP, preferences, delete account |
| `template.controller.ts` | `/templates/*` | Browse, create, publish, autofill, image gen |
| `instance.controller.ts` | `/instances/*` | Create/list/archive/reset/delete instances, protagonist, settings |
| `persona.controller.ts` | `/personas/*` | Player persona CRUD |
| `chronicle.controller.ts` | `/chronicle/*` | Events, memories, recap, calendar, bonds, places, threads, side chats, edit/replay/rewind |
| `play-ws.controller.ts` | `/ws/play` | WebSocket upgrade and message dispatch |
| `admin.controller.ts` | `/admin/*` | Admin CRUD, projections inspect, continuity audit |

---

## Services (domain logic)

### Core gameplay

| Service | File | What it does |
|---------|------|--------------|
| **Generation** | `generation.service.ts` | **Thin dispatch** — validates session, enqueues `generate` / `replay` / `side_chat` jobs. Loads Play bootstrap (recent events, memories, codex). Context packet is built in the **worker**, not here. |
| **Play WebSocket** | `play-ws.service.ts` | Socket registry, Redis pub/sub relay, generation locks, rate limits, forwards actions to generation service |
| **Context packet** | `context-packet.service.ts` | Assembles one turn's briefing: recents → RAG → codex pins → time/place. Also `buildSideChatPacket` for private chats |
| **Memory** | `memory.service.ts` | Chronicle reads (events, memories, recap, threads), edit/replay/rewind, projection rebuild, `mainVisibleMemoryScope` privacy gate |
| **Character codex** | `character-codex.service.ts` | Apply deltas, rank for injection, seed protagonist, rebuild from ledger on rewind, relationship meters |
| **Entity graph** | `entity-graph.service.ts` | Entity registry, edges, location facts, mention resolution, rewind repair |
| **Memory supersession** | `memory-supersession.service.ts` | When codex retires a fact, find and archive matching memory vectors |

### Retrieval

| Service | File | What it does |
|---------|------|--------------|
| **RAG** | `providers/rag.provider.ts` | Hybrid search: vector + keyword + entity neighborhood + location + timeline filter + RRF fusion. `querySummaries` for distant chapter recall |

### Time, place, side chat

| Service | File | What it does |
|---------|------|--------------|
| **Time** | `time.service.ts` | Story calendars, timeline branches, time anchors, fork/switch reality, flashback re-anchor |
| **Location** | `location.service.ts` | Places index + per-place journal (read-only; cursor writes happen in generation worker) |
| **Side chat** | `side-chat.service.ts` | List threads + paginated history per character |

### Instance & template lifecycle

| Service | File | What it does |
|---------|------|--------------|
| **Instance** | `instance.service.ts` | CRUD, tier limits, **Redis session cache** (`session:{id}`, 1h TTL), protagonist, settings |
| **Template** | `template.service.ts` | Template CRUD, publish, embed lore to Pinecone |
| **Deletion** | `deletion.service.ts` | Cascade delete/reset: Mongo + Pinecone namespaces + Redis session bust |
| **Persona** | `persona.service.ts` | Persona CRUD; busts sessions on instances using deleted persona |

### Quality & ops

| Service | File | What it does |
|---------|------|--------------|
| **Continuity audit** | `continuity-audit.service.ts` | Read-only 8-check consistency report across projections |
| **Admin** | `admin.service.ts` | Dashboard, paginated CRUD, event projection inspect, drift status listing |
| **Auth** | `auth.service.ts` | Users, passwords, JWT payload, preferences |
| **Autofill** | `autofill.service.ts` | One-shot LLM world/character draft for Forge |
| **Image** | `image.service.ts` | Cover/avatar generation → S3 |
| **Storage** | `storage.service.ts` | S3 upload/delete/promote |
| **Pinecone cleanup** | `pinecone-cleanup.service.ts` | Namespace/vector deletion helpers |

### External providers

| Provider | File | What it does |
|----------|------|--------------|
| **Auth provider** | `providers/auth.provider.ts` | Google token verify, Twilio OTP (mock in dev) |

---

## Prompt & AI utilities (not services, but hot path)

| Module | File | Role |
|--------|------|------|
| Prompt builder | `utils/prompt-builder.ts` | Static cacheable prefix + dynamic sections + token budgets + recents floor |
| Prose hygiene | `utils/prose-hygiene.ts` | Rules-first validation; LLM repair on failure |
| Token counter | `utils/token-counter.ts` | Cached tiktoken for budget math |
| Event windows | `utils/event-window.ts` | Shared limits: 6 prompt recents, 20 Play load, 50 Chronicle page |
| AI client | `ai/client.ts` | `callLLM`, `callLLMStream` with timeout/abort |
| AI models | `ai/models.ts` | Central model ID map + env overrides |

---

## Key design choices

1. **Dispatch is thin** — RAG and codex assembly moved to worker (`context-packet.service`) so retrieval runs before codex selection.
2. **Session cache** — Redis avoids re-assembling instance JSON every WS message; busted on rewind/settings/persona changes.
3. **Fire-and-forget on hot path** — Generation worker defers location facts, codex compaction, supersession, generation logs so streaming isn't blocked.
4. **Projection rebuild** — Edits, replays, rewinds go through `memory.service` + `entity-graph.service` to keep derived state consistent with events.

---

## Adding a new feature

1. Model types in `src/models/` (+ index in `mongo-indexes.ts` if new queries)
2. Service method with ownership checks (`worldInstances().findOne({ _id, player_id })`)
3. Controller handler
4. Route in appropriate `*.routes.ts`
5. If async/LLM-heavy → enqueue in `src/queues/` + processor in `worker/processors/`
6. Update [API.md](./API.md) and [code-reference/SERVER.md](../code-reference/SERVER.md)
