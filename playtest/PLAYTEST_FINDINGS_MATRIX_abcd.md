# Regression Matrix — runs a → b → c → d (2026-06-12)

One row per cluster/finding, status across the four 6-agent runs. Built from the
**reviewer-verified** data (DB/API spot-checks), not raw agent self-reports — see the
verification addenda in each merged report. Sources:
`PLAYTEST_FINDINGS_MERGED_2026-06-12{,b,c,d}.md`. Supersedes `…MATRIX_abc.md`.

Legend: 🔴 broken · 🟡 partial/split · 🟢 fixed/held · ⬛ not-yet-built/n-a · ✳️ NEW this run

| # | Cluster / finding | a | b (fix-1) | c (fix-2) | d (fix-3) | Verdict |
|---|---|---|---|---|---|---|
| I-a | Event-edit wrong field → 400 | 🔴 | 🟢 | 🟢 | 🟢 | **CLOSED** |
| I-b | Event-edit unchanged/empty → 400 | 🔴 | 🟡 | 🟢 | 🟢 | **CLOSED** |
| L | Bonds shows companion | 🔴 | 🟢 | 🟢 | 🟢 | **CLOSED** |
| N7 | Calendar `month_names` serializer | ⬛ | 🔴✳️ | 🟢 | 🟢 | **CLOSED** |
| G | Recap `recap_text`/`when`/`current_place` | 🔴 | 🔴 | 🟢 | 🟢 | **CLOSED** |
| E | Supersession symmetric (retire+link) | 🔴 | 🟡 | 🟢 | 🟢 | **CLOSED** |
| N2 | Dangling `updates_memory_ids` on rewind | ⬛ | 🔴✳️ | 🟢 | 🟢 | **CLOSED** |
| N5 | Sentient AI present every turn | ⬛ | 🔴✳️ | 🟢 | 🟢 | **CLOSED** |
| H | `location_state` positive transforms | 🔴 | 🔴 | 🟢 | 🟢 | **CLOSED** |
| D-loop | Travel self-loop / null world-root | ⬛ | 🔴✳️ | 🟢 | 🟢 | **CLOSED** (#4 Veil direct destination) |
| **A** | Side-chat secret leak (sentient) | 🔴 | 🔴 | 🟡 | 🟢 | **CLOSED** — d #3/#4 PASS via codex-card scrub + `audit:side-chat-privacy` |
| **R2** | Codex merge on name correction | — | — | 🔴✳️ | 🟢 | **CLOSED** — d #1 Mara Chen single card |
| **B2a** | Don't card the player | ⬛ | 🟢 | 🔴 | 🟢 | **CLOSED (d)** — root cause was the **un-awaited alias persist racing later turns** (NOT a `kind:character` gap; Lena is `is_sentient:true`). Persist now awaited inside the turn lock. |
| **B2b** | Don't card absent relatives | 🔴 | 🔴 | 🟡 | 🟡 | **PARTIAL** — d #2/#4/#6 PASS; **#5 Mira regressed** |
| **B1** | Identity attribution (player↔AI) | 🔴 | 🔴 | 🟡 | 🟡 | **PARTIAL** — d #3 badge held; **#6 "shiver"→Elara**, #4 NSFW-mem attribution |
| **D** | Location dedup (article/parent variants) | 🔴 | 🔴 | 🔴 | 🔴 | **STILL OPEN** — all spatial lanes (d #2 Wildwood ×3→×2 only) |
| D-cursor | Cursor lag/reset on /continue & returns | 🔴 | 🔴 | 🔴 | 🔴 | **STILL OPEN** — d #1 seq 8/18–19; #6 return improved |
| **N1** | Orphan memory after rewind+edit (existing) | ⬛ | 🔴✳️ | 🔴 | 🔴 | **STILL OPEN** — existing `6a2bd590…` (forward-only fix; needs data repair) |
| **N3** | Rewind re-mints stale codex canonical name | ⬛ | 🔴✳️ | 🔴 | 🔴 | **STILL OPEN** — existing `6a2bd56e…` sister still "Mara" |
| **N4-NSFW** | Explicit prompt routes to SFW model | ⬛ | 🔴✳️ | 🔴 | 🔴 | **STILL OPEN** — `gemma` on explicit (#4) |
| R4 | `/continue season` prose ignores themed months | — | 🟡 | 🔴✳️ | 🔴 | **STILL OPEN** — #2 autumn/winter vs Glimmerfall |
| Merchant | Vague passer-by carded | 🔴 | 🟢 | — | 🔴 | **OPEN again** — d #6 Merchant carded (variance/heuristic) |
| C | Presence: recall vs co-location | 🔴 | 🔴 | 🟡 | 🟡 | **PARTIAL** — d #3 Mira present while trapped; #5 Swapnil drops |
| A-gm | "Echoes returns side_chat" on GM lanes | 🟡 | 🔴 | 🟢* | ⬛ | **NOT A BUG** (by-design). Do not work. |
| K | Branch-timeline turn hang | 🔴✳️ | 🟡 | ⬛ | ⬛ | not retested — intermittent |
| J1 | Replay POV swap (companion) | ⬛ | 🔴✳️ | 🟢 | ⬛ | not reproduced — variance |

## How to read this
- **13 clusters CLOSED and holding** through d (top block). Fix batch 3 closed the
  hardest two — **A (side-chat codex-card leak)** and **R2 (codex merge on correction)** —
  plus **B2a** once the real mechanism (not the reported one) was found.
- **B2a was misdiagnosed in the run-d report** as a `kind:character` coverage gap. The
  data shows Lena is `is_sentient:true`, so the guard *did* run; the card was minted by a
  later turn because the self-intro alias was persisted **fire-and-forget** and hadn't
  landed when the next extraction read the player entity. Fix = await the persist inside
  the per-instance turn lock. Detector regression case added to `audit:side-chat-privacy`.
- **The stubborn live-corruption core is now D (location) + B1 (residual) + B2b/Merchant
  (carding heuristics) + N1/N3 (existing-save rewind) + NSFW routing.** These are the
  semantic/LLM-path bugs structural audits never catch.

## Active fix queue (post-d → addressed in FIX BATCH 4)
See `FIX_BATCH_4_2026-06-12.md`. Status after batch 4 (awaiting run-e live confirmation):
1. **D location dedup** — ✅ forward fix (route memory mentions through `resolveLocationAnchor`). **D-cursor** — deliberate non-change (accepted ambiguity; avoids phantom-travel regression).
2. **B1 residual** — ✅ prompt-layer fix (anti-fabrication + first-person-sensation boundary). Semantic; verify live.
3. **B2b + Merchant carding** — ✅ deterministic guards (absent-relative via narration; bare-descriptor passer-by). `audit:carding-routing` green.
4. **N1 / N3** — ✅ N1 forward fix (surviving-memory ref prune) + data repair both saves (N1 0 orphans; N3 canonical=Mira).
5. **N4-NSFW routing** — ✅ ambiguous-pattern tier in classifier; `audit:carding-routing` green.
6. **Still untouched:** R4 calendar prose grounding; C presence (#3/#5).

**Do NOT work:** A-gm Echoes (by-design).

## Existing-data repair — DONE this pass (forward-only fixes don't heal old saves)
- Leaked **Jora** side-chat card (`6a2bea89…`): stripped `immutable_facts`/`mutable_state`/`hidden_thought`.
- Player self-intro cards merged into the canonical `The Player` entity + deleted:
  **Swapnil** (`6a2bea92…`), **Kael** (`6a2bea8c…`), **Alex** (`6a2bea91…`), and the run-d
  **Swapnil** on Lena (`6a2bfb12…`). Zero dangling memory→entity refs; sessions busted.
- **Still un-repaired:** N1 orphan (`6a2bd590…`) and N3 stale canonical (`6a2bd56e…`) on the
  older keeper-ish saves — queued under fix #4.
