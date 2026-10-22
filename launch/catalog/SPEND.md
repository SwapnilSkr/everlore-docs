# Launch catalog: spend ledger

Running total of what publishing the catalog has cost. Measured, not estimated: the provider's own usage counter (`GET openrouter.ai/api/v1/key`, `data.usage`, USD) is read on prod immediately before and after each run. Lore embeddings go to OpenAI directly and are too small to show on a bill (about $0.00002 a world); they are noted, not counted. Covers cost nothing (Codex app, inside the ChatGPT plan). ₹ at 84 per $.

| Date | Run | Worlds written | Counter before | Counter after | Spent |
|---|---|---|---|---|---|
| 2026-10-22 | romance, first apply (stopped: index namespace limit) | 1 published, 1 left as a draft | $87.228110 | $87.229324 | $0.0012 (₹0.10) |
| 2026-10-22 | romance, second apply (after freeing 38 leftover namespaces) | 23 created, 1 draft published | $87.229324 | $87.241913 | $0.0126 (₹1.06) |
| 2026-10-22 | romance, world-guide rewrite (all 25 updated, casts re-read) | 25 updated | $87.241913 | $87.256144 | $0.0142 (₹1.20) |

**Running total: $0.0280 (₹2.36)** for 25 live worlds, about $0.0011 (₹0.09) a world including one full rewrite. Cap agreed by the owner for the romance batch: ₹50.

## Memory move (one vector section), not the catalog

Re-embedding sample messages for the old-vs-new recall comparison (OpenAI direct, $0.02 per 1M tokens). Counted separately from the catalog.

| Date | Run | Tokens | Spent |
|---|---|---|---|
| 2026-10-22 | recall comparison, first pass | 6,404 | $0.000128 (₹0.01) |
| 2026-10-22 | recall comparison, with breakdown | 6,404 | $0.000128 (₹0.01) |
| 2026-10-22 | exact-score check (both layouts graded against exact similarity) | 6,404 | $0.000128 (₹0.01) |
| 2026-10-22 | trial switch: four recall timing runs (about 720 short embeddings, estimated) | ~11,000 | ~$0.00022 (₹0.02) |

**Total: about $0.0006 (₹0.05).**
