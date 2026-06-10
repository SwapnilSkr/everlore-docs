# What's Open Next

Honest gaps, reopened checklist items, and recommended order — so you know where planning picks up.

---

## Phases marked complete but with honest gaps

### Live-turn verification (recommended before big new features)

Several paths are **code-reviewed and unit-tested** but not confirmed on a real LLM generation against a running server + worker + app:

| Path | What to verify |
|------|----------------|
| Side chat | Bonds → private chat streams; doesn't leak to Echoes/Timeline |
| Travel | Narrated move creates `travel` event + calendar advance |
| Location facts | Place journal shows state/facts after relevant turn |
| Metadata extraction | `time_elapsed`, `present_characters`, end location |

**How:** Start server, worker, app; play one focused scenario each; check Lore Tome + admin projections.

---

## Reopened checklist items (intentional next planning)

### Phase 4 — Broader BM25 🔄

**Current state:** Keyword search runs Mongo `$text` on memory `text`, `subjects`, `objects`.

**Reopened goal:** Also index/search entity names, place names, promise labels, event labels.

**Questions before building:**
- Echoes search only, or prompt RAG too?
- How to measure improvement?
- How to avoid polluting side-chat privacy boundaries?

**Files to touch:** `mongo-indexes.ts`, `rag.provider.ts`, maybe `memory.service.getMemories`

---

### Phase 2 — Memory version links 🔄

**Current state:** Supersession and dedup work at vector/text level. No explicit graph between memory atoms.

**Reopened goal:** `updates_memory_ids`, `extends_memory_ids`, `derives_from_memory_ids`

**Questions before building:**
- Who writes links — extractor, dedup job, supersession service?
- Admin/explain UI or retrieval prompt use?
- Backfill strategy for existing memories?

**Files to touch:** `memory.model.ts`, `memory.processor.ts`, `memory-supersession.service.ts`, admin projections

---

## Deferred by design (don't build until triggered)

| Item | Trigger to reopen |
|------|-------------------|
| Phase 1 revision counters | A feature needs revision-addressable projections |
| Phase 9 cold archival | Real Mongo storage pressure or cost data |
| Full proportional context packet allocator v2 | Only if current token budgets prove insufficient |

---

## Micro-optimizations noted but not done

| Opt | Benefit |
|-----|---------|
| Share embed between `queryRag` + `querySummaries` | −1 embedding call/turn |
| Dedup clustering instead of O(n²) | Faster weekly job at scale |
| `scene_summaries` index on `end_sequence` | Minor query perf |

---

## Suggested planning order (from HANDOFF.md)

```text
1. Live-turn verification pass     ← confidence before changing retrieval
2. Discuss Phase 4 broader BM25    ← define scope + metrics first
3. Discuss Phase 2 memory links    ← schema + lifecycle design
4. Revisit deferred items only when a concrete consumer appears
```

---

## What you have vs original vision doc

| Vision (`memory/MEMORY_ARCHITECTURE.md`) | Reality today |
|------------------------------------------|---------------|
| Full entity graph with calendar/timeline entity types | Graph exists; calendar/timeline are separate models, not entity types |
| Memory `updates/extends/derives` graph | Not built (reopened) |
| Explicit context packet object in API | Built internally in worker, not exposed to clients |
| Side-character scenes elsewhere in world time | Side chats don't advance main calendar (by design) |
| Semantic summary retrieval | ✅ Shipped |
| Player inspects full graph | Partial — Bonds, Places, Echoes, admin projections |

---

## For your next AI chat session

If continuing the build, point the agent at:

1. This folder: `everlore-docs/system-guide/`
2. Task list: `everlore-docs/memory/CHECKLIST.md`
3. Agent handoff: `everlore-docs/memory/HANDOFF.md`

**Prompt starter:**

> Read `everlore-docs/system-guide/README.md` and `memory/HANDOFF.md`. Phases 1–10 are complete. Phase 4 (broader BM25) and Phase 2 (memory version links) are reopened. Start with [live verification / Phase 4 planning / Phase 2 design — pick one]. Do not rewrite shipped architecture; extend it.

---

## Quick reference: where to change what

| You want to change… | Start here |
|---------------------|------------|
| What the AI sees each turn | `context-packet.service.ts`, `prompt-builder.ts` |
| How memories are extracted | `memory.processor.ts` |
| How characters update | `character-codex-extractor.ts`, `character-codex.service.ts` |
| Search/recall quality | `rag.provider.ts` |
| Rewind behavior | `memory.service.ts`, `entity-graph.service.ts` |
| Play UI | `everlore/lib/features/play/` |
| Lore Tome tabs | `everlore/lib/features/chronicle/` |
| Privacy rules | `rag.provider.ts`, grep `side_chat` exclusions |
| Background jobs | `worker/processors/maintenance.processor.ts`, `worker/index.ts` |
