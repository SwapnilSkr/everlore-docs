# Regression Matrix — runs a → b → c (2026-06-12)

One row per cluster/finding, status across the three 6-agent runs. Built from the
**reviewer-verified** data (DB/API spot-checks), not raw agent self-reports — see the
verification addenda in each merged report. Sources:
`PLAYTEST_FINDINGS_MERGED_2026-06-12{,b,c}.md`.

Legend: 🔴 broken · 🟡 partial/split · 🟢 fixed/held · ⬛ not-yet-built/n-a · ✳️ NEW this run

| # | Cluster / finding | a (first) | b (post fix-1) | c (post fix-2) | Verdict |
|---|---|---|---|---|---|
| I-a | Event-edit wrong field → 400 | 🔴 | 🟢 | 🟢 | **CLOSED** |
| I-b | Event-edit unchanged/empty → 400 | 🔴 | 🟡 | 🟢 | **CLOSED** (b partial; c exact-copy PASS) |
| L | Bonds shows companion | 🔴 | 🟢 | 🟢 | **CLOSED** |
| N7 | Calendar `month_names` serializer | ⬛ | 🔴✳️ | 🟢 | **CLOSED** |
| G | Recap `recap_text`/`when`/`current_place` | 🔴 | 🔴 | 🟢 | **CLOSED** |
| E | Supersession symmetric (retire+link) | 🔴 | 🟡 | 🟢 | **CLOSED** (verified: old atom `status:superseded`+`superseded_by`) |
| N2 | Dangling `updates_memory_ids` on rewind | ⬛ | 🔴✳️ | 🟢 | **CLOSED** |
| N5 | Sentient AI present every turn | ⬛ | 🔴✳️ | 🟢 | **CLOSED** |
| H | `location_state` positive transforms | 🔴 | 🔴 | 🟢 | **CLOSED** (#4 heal captured) |
| D-loop | Travel self-loop / null world-root | ⬛ | 🔴✳️ | 🟢 | **CLOSED** (#4 no `from==to`, names non-null) |
| **B1** | Identity attribution (player↔AI) | 🔴 | 🔴 | 🟡 | **PARTIAL** — #3/#5 fixed; #4/#6 residual AI-subject |
| **B2a** | Don't card the player | ⬛ | 🟢(b) | 🔴✳️ | **REGRESSED** — player self-name now cards (Alex/Swapnil/Kael) alongside `The Player` entity |
| **B2b** | Don't card absent relatives | 🔴 | 🔴 | 🟡 | **PARTIAL** — fixed fresh #5/#6; fails #2/#4 |
| **A** | Side-chat secret leak (sentient) | 🔴 | 🔴 | 🟡 | **PARTIAL** — #3 gated ✅; **#4 REAL leak via narration prompt** (not curation) |
| A-gm | "Echoes returns side_chat" on GM lanes | 🟡 | 🔴 | 🟢* | **NOT A BUG** (by-design: GM protagonist is a knower). Discounted. |
| **D** | Location dedup (article/parent variants) | 🔴 | 🔴 | 🔴 | **STILL OPEN** — all spatial lanes |
| D-cursor | Cursor lag/reset on /continue & returns | 🔴 | 🔴 | 🔴 | **STILL OPEN** |
| **N1** | Orphan memory after rewind+edit (existing) | ⬛ | 🔴✳️ | 🔴 | **STILL OPEN** — existing `6a2bd590…` audit still FAIL |
| **N3** | Rewind re-mints stale codex canonical name | ⬛ | 🔴✳️ | 🔴 | **STILL OPEN** — #3 sister still "Mara" |
| **N4-NSFW** | Explicit prompt routes to SFW model | ⬛ | 🔴✳️ | 🔴 | **STILL OPEN** — `gemma` on explicit (#4) |
| C | Presence: recall vs co-location | 🔴 | 🔴 | 🟡 | **PARTIAL** — phantom voices #1; Alex in present #3 |
| K | Branch-timeline turn hang | 🔴✳️ | 🟡 | ⬛ | not retested c — intermittent |
| J1 | Replay POV swap (companion) | ⬛ | 🔴✳️ | 🟢 | not reproduced b→c — likely OK / variance |
| R2 | Mara+Mira dual codex card (memory superseded, codex not merged) | — | — | 🔴✳️ | **NEW c** — codex merge missing on name correction |
| R4 | `/continue season` prose ignores themed months | — | 🟡 | 🔴✳️ | **NEW/again** — calendar prose grounding |

\* A-gm shows 🟢 in c only in the sense that it's confirmed by-design; the agents still
filed it — it should not be worked.

## How to read this
- **8 clusters CLOSED and holding** across c (top block) — the deterministic + serializer
  + supersession + recap work is solid; don't regress it.
- **The live corruption core is stubborn:** B1, B2a/b, A(sentient), D, C — the
  semantic/LLM-path bugs. These are the ones structural audits never catch.
- **The fix batches keep introducing rewind-class bugs** (N1/N2/N3 appeared in b; N2
  closed, N1/N3 still open) — rewind + curation/supersession/codex is the fragile seam.
- **Whack-a-mole watch:** B2a went 🟢(b) → 🔴(c) — a fix in b regressed in c. Re-verify
  B2a/B2b every run.

## Active fix queue (post-c, verified)
1. Sentient side-chat leak via the **generation prompt/recents/RAG** path (#4).
2. Player codex card (self-intro) + B1 residual AI-subject inversions (#4/#6).
3. Location dedup + cursor lag (D) — chronic, unfixed across all 3 runs.
4. N1 orphan repair + N3 rewind canonical name (rewind seam).
5. NSFW routing (N4-NSFW).
6. Cleanups: codex merge on name correction (R2), B2b relatives (#2/#4), calendar prose grounding (R4).
**Do NOT work:** N6 Echoes-GM (by-design).
