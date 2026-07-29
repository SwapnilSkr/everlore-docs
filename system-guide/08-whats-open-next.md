# What's Open Next

Honest gaps and recommended order — so you know where planning picks up.

**Last reconciled with playtest matrix:** July 30, 2026 (docs sync). Live open-bug status is **not** duplicated here — use the playtest sources of truth below.

---

## Open bugs & live regressions (source of truth)

Do **not** invent closed/open status from this file alone.

| Doc | Role |
|-----|------|
| [`../playtest/PLAYTEST_FINDINGS_MATRIX_abcd.md`](../playtest/PLAYTEST_FINDINGS_MATRIX_abcd.md) | Cluster status across runs a→d + active queue notes |
| [`../playtest/FIX_BATCH_4_2026-06-12.md`](../playtest/FIX_BATCH_4_2026-06-12.md) | Forward fixes for the post-d open core (await live confirmation where noted) |

Read the matrix verdict column and the FIX_BATCH_4 scorecard before picking a bug to work.

---

## Shipped since the original “reopened planning” note

### Phase 2 — Memory version links (Slice 1) ✅

**Shipped:** `updates_memory_ids` (+ `superseded_by_event_ids` marks) from the supersession → curator path; rewind/edit prune; `repair_memory_links` maintenance; admin projections (knowledge-gated).

**Still deferred (no producer):** Slice 2 `extends_memory_ids`, Slice 3 `derives_from_memory_ids`.

Details / commits: [`../memory/HANDOFF.md`](../memory/HANDOFF.md) State bullet for Phase 2 Slice 1.

### Product surfaces also shipped (docs sync)

- **World Actions** in Play (continue / time / travel / kinship / relation-candidate review)
- **Membership & Ink** (`/membership`, `/billing/*`, reserve → settle on turns)
- Kinship graph + relation-candidate review APIs
- Post-prose entity adjudication + `signal_ledger` instrumentation

---

## Still open for planning (not bugs)

### Phase 4 — Broader BM25 🔄 (deferred with gate)

**Current state:** Keyword search runs Mongo `$text` on memory `text`, `subjects`, `objects`.

**Goal:** Also index/search entity names, place names, promise labels, event labels — **only after** the measurement gate in [`../memory/RETRIEVAL_MEASUREMENT.md`](../memory/RETRIEVAL_MEASUREMENT.md) (do not run the eval on tiny instances).

**Files to touch when triggered:** `mongo-indexes.ts`, `rag.provider.ts`, maybe `memory.service.getMemories`

---

## Deferred by design (don't build until triggered)

| Item | Trigger to reopen |
|------|-------------------|
| Phase 2 Slices 2–3 (`extends` / `derives_from`) | A real producer + consumer design |
| Phase 1 revision counters | A feature needs revision-addressable projections |
| Phase 9 cold archival | Real Mongo storage pressure or cost data |
| Full proportional context packet allocator v2 | Only if current token budgets prove insufficient |
| Location Graph P2.5 / P3 / open-world limits | Content-gated — see `LOCATION_GRAPH.md` |

---

## Micro-optimizations noted but not done

| Opt | Benefit |
|-----|---------|
| Share embed between `queryRag` + `querySummaries` | −1 embedding call/turn |
| Dedup clustering instead of O(n²) | Faster weekly job at scale |
| `scene_summaries` index on `end_sequence` | Minor query perf |

---

## Suggested planning order

```text
1. Playtest matrix / FIX_BATCH_4 leftovers ← live semantic bugs
2. Discuss Phase 4 broader BM25 only at scale gate
3. Phase 2 Slices 2–3 only with a concrete producer
4. Revisit deferred items only when a concrete consumer appears
```

---

## What you have vs original vision doc

| Vision (`memory/MEMORY_ARCHITECTURE.md`) | Reality today |
|------------------------------------------|---------------|
| Full entity graph with calendar/timeline entity types | Graph exists; calendar/timeline are separate models |
| Memory `updates/extends/derives` graph | **`updates` Slice 1 shipped**; extends/derives deferred |
| Explicit context packet object in API | Built internally in worker, not exposed to clients |
| Side-character scenes elsewhere in world time | Side chats don't advance main calendar (by design) |
| Semantic summary retrieval | ✅ Shipped |
| Player inspects full graph | Partial — Bonds, Places, Echoes, World Actions kinship, admin projections |

---

## For your next AI chat session

If continuing the build, point the agent at:

1. This folder: `everlore-docs/system-guide/`
2. Open bugs: `everlore-docs/playtest/PLAYTEST_FINDINGS_MATRIX_abcd.md` + `FIX_BATCH_4_2026-06-12.md`
3. Task list: `everlore-docs/memory/CHECKLIST.md`
4. Agent handoff: `everlore-docs/memory/HANDOFF.md`

**Prompt starter:**

> Read `everlore-docs/system-guide/README.md` and `memory/HANDOFF.md`. Phases 1–10 are complete. Phase 2 Slice 1 (`updates_memory_ids`) shipped; Phase 4 BM25 remains gated. Open live bugs → playtest matrix + FIX_BATCH_4. Start with [matrix cluster / BM25 gate / … — pick one]. Do not rewrite shipped architecture; extend it.

---

## Quick reference: where to change what

| You want to change… | Start here |
|---------------------|------------|
| What the AI sees each turn | `context-packet.service.ts`, `prompt-builder.ts` |
| How memories are extracted | `memory.processor.ts` |
| How characters update | `character-codex-extractor.ts`, `character-codex.service.ts` |
| Kinship / candidates | `kinship-graph.service.ts`, `relation-candidate.service.ts`, `worker/lib/kinship-*` |
| Search/recall quality | `rag.provider.ts` |
| Rewind behavior | `memory.service.ts`, `entity-graph.service.ts` |
| Billing / Ink | `billing.service.ts`, `play-ws.service.ts`, worker settle paths |
| Play UI / World Actions | `everlore/lib/features/play/` |
| Lore Tome tabs | `everlore/lib/features/chronicle/` |
| Membership UI | `everlore/lib/features/billing/` |
| Privacy rules | `rag.provider.ts`, grep `side_chat` exclusions |
| Background jobs | `worker/processors/maintenance.processor.ts`, `worker/index.ts` |
