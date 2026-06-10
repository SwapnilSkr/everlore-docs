# Everlore System Guide

**A plain-language walkthrough of everything that was built.**

This folder explains the full Everlore memory system as it exists today — not the original plan, but what actually shipped in code across the Flutter app, Bun server, and background workers.

**Last updated:** June 10, 2026 (Phases 1–10 complete; Phase 2 memory-version links and Phase 4 broader BM25 reopened for planning)

---

## Who this is for

You want to understand:

- What happens when you play a turn (frontend → server → AI → memory)
- Where each button and tab lives, and when it appears
- How the backend remembers a 10,000-turn story without sending 10,000 turns to the AI
- What each of the 10 memory phases actually means in practice
- What best practices were used and what could break at scale

You do **not** need to read every source file. Start here, then drill into the topic you care about.

---

## How to read these docs

| Doc | Read this if you want to understand… |
|-----|--------------------------------------|
| [01 — Core philosophy](01-core-philosophy.md) | The *why*: events vs projections, bounded prompts, privacy rules |
| [02 — One turn, start to finish](02-one-turn-journey.md) | The *flow*: tap Send → streamed reply → memories saved |
| [03 — Memory layers](03-memory-layers-explained.md) | Events, memories, codex, summaries, entity graph, time, places |
| [04 — Frontend map](04-frontend-where-things-live.md) | Play screen, Lore Tome tabs, side chats — what appears when |
| [05 — Backend & workers](05-backend-and-workers.md) | Services, queues, databases, APIs |
| [06 — All ten phases](06-all-ten-phases.md) | Checklist phases translated into shipped features + file paths |
| [07 — Scaling & risks](07-scaling-risks-and-practices.md) | What stays flat vs what grows, known bottlenecks |
| [08 — What's open next](08-whats-open-next.md) | Reopened items, honest gaps, recommended next work |
| [09 — Visual maps](09-visual-maps.md) | ASCII diagrams: Play layout, Lore Tome tabs, data flow |

## Code reference

| Doc | Purpose |
|-----|---------|
| [../code-reference/SERVER.md](../code-reference/SERVER.md) | Every server `.ts` file — purpose + optimizations |
| [../code-reference/CLIENT.md](../code-reference/CLIENT.md) | Every `lib/` Dart file — purpose + patterns |

---

## Repos (three separate git repos)

| Folder | What it is |
|--------|------------|
| `everlore/` | Flutter app (Play, Lore Tome, onboarding) |
| `everlore-server/` | API + WebSocket + BullMQ workers |
| `everlore-docs/` | This documentation |

The parent `rpg-ai/` folder is **not** a git repo.

---

## One-sentence summary

**The full story lives forever in Mongo; the AI only ever sees a small, ranked briefing assembled each turn from memories, character sheets, summaries, and the last few messages — and the app gives you tools to read, edit, rewind, and explore that world.**

---

## Related docs (deeper / planning)

| Location | Purpose |
|----------|---------|
| `memory/CHECKLIST.md` | Authoritative task list with commit refs |
| `memory/HANDOFF.md` | Agent handoff — how to continue the build |
| `memory/MEMORY_ARCHITECTURE.md` | Target infinite-memory vision |
| `server/MEMORY_ARCHITECTURE.md` | Current server memory stack (codex + RAG detail) |
