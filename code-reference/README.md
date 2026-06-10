# Code Reference

File-by-file map of the Everlore codebase: **what each file does**, in simple terms, plus **performance patterns** used.

| Doc | Repo | Files covered |
|-----|------|---------------|
| [SERVER.md](./SERVER.md) | `everlore-server/` | ~103 TypeScript files (config, services, workers, …) |
| [CLIENT.md](./CLIENT.md) | `everlore/` | ~98 Dart files under `lib/` |

**Not a substitute for reading code** — use this to orient before diving into a file.

**Product/system context:** [../system-guide/README.md](../system-guide/README.md)

---

## How to read these docs

Each row has:

- **Purpose** — what the file is responsible for
- **Key exports** — main functions/classes
- **Optimizations** — caching, batching, bounds, async patterns (blank if none)

---

## Cross-cutting themes (server)

| Pattern | Where |
|---------|-------|
| Redis session cache (1h) | `instance.service.ts` |
| Mongo projection on reads | RAG, context packet, admin, chronicle |
| `Promise.all` parallel fetches | context packet, RAG arms, continuity audit |
| Fire-and-forget post-turn | generation processor (location facts, compaction, logs) |
| BullMQ offload | memory, summaries, maintenance |
| Token / count budgets | `prompt-builder.ts`, `context-packet.service.ts`, `event-window.ts` |
| Bounded windows | 6 prompt recents, 20 Play load, 50 client memories |
| Deterministic Pinecone ids | summaries (rebuild overwrites same vector) |
| Fail-closed privacy | `rag.provider.ts`, `mainVisibleMemoryScope` |

## Cross-cutting themes (client)

| Pattern | Where |
|---------|-------|
| WsManager singleton | one WS, reconnect, offline queue |
| Play state trim | 100 events / 50 memories |
| SQLite event cache | 200 turns, fast resume |
| Lazy Chronicle tab loads | fetch on first visit |
| Optimistic UI | memory edit/delete, play turn placeholder |
| Stream reveal + watchdogs | PlayCubit generation/replay |
