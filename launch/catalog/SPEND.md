# Launch catalog: spend ledger

Running total of what publishing the catalog has cost. Measured, not estimated: the provider's own usage counter (`GET openrouter.ai/api/v1/key`, `data.usage`, USD) is read on prod immediately before and after each run. Lore embeddings go to OpenAI directly and are too small to show on a bill (about $0.00002 a world); they are noted, not counted. Covers cost nothing (Codex app, inside the ChatGPT plan). ₹ at 84 per $.

| Date | Run | Worlds written | Counter before | Counter after | Spent |
|---|---|---|---|---|---|
| 2026-10-22 | romance, first apply (stopped: index namespace limit) | 1 published, 1 left as a draft | $87.228110 | $87.229324 | $0.0012 (₹0.10) |
| 2026-10-22 | romance, second apply (after freeing 38 leftover namespaces) | 23 created, 1 draft published | $87.229324 | $87.241913 | $0.0126 (₹1.06) |
| 2026-10-22 | romance, world-guide rewrite (all 25 updated, casts re-read) | 25 updated | $87.241913 | $87.256144 | $0.0142 (₹1.20) |
| 2026-10-22 | romance, covers re-uploaded compressed (no text changed) | 25 updated | $87.256144 | $87.256144 | $0.0000 |
| 2026-10-22 | epic, first apply | 25 created | $87.256144 | $87.269472 | $0.0133 (₹1.12) |
| 2026-10-22 | dark, first apply | 20 created | $87.269472 | $87.279195 | $0.0097 (₹0.82) |
| 2026-10-22 | horror, first apply | 25 created | $87.279850 | $87.291823 | $0.0120 (₹1.01) |

**Running total: $0.0630 (₹5.29)** for 95 live worlds, about $0.0007 (₹0.06) a world. Romance $0.0280 (₹2.36) including one full rewrite; Epic $0.0133 (₹1.12); Dark $0.0097 (₹0.82); Horror $0.0120 (₹1.01). Cap agreed by the owner for the romance batch: ₹50.

## Memory move (one vector section), not the catalog

Re-embedding sample messages for the old-vs-new recall comparison (OpenAI direct, $0.02 per 1M tokens). Counted separately from the catalog.

| Date | Run | Tokens | Spent |
|---|---|---|---|
| 2026-10-22 | recall comparison, first pass | 6,404 | $0.000128 (₹0.01) |
| 2026-10-22 | recall comparison, with breakdown | 6,404 | $0.000128 (₹0.01) |
| 2026-10-22 | exact-score check (both layouts graded against exact similarity) | 6,404 | $0.000128 (₹0.01) |
| 2026-10-22 | trial switch: four recall timing runs (about 720 short embeddings, estimated) | ~11,000 | ~$0.00022 (₹0.02) |
| 2026-10-22 | final switch: one recall timing run (estimated) | ~2,700 | ~$0.00005 (₹0.005) |

**Total: about $0.0007 (₹0.06).** The move itself (copy, check, removal) used no paid calls.
