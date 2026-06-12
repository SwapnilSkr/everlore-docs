# Merged Playtest Findings — 2026-06-12d (post fix batch 3)

Synthesis of the **fourth** 6-agent run. Diffs against
`PLAYTEST_FINDINGS_MERGED_2026-06-12c.md`. Per-agent sources:
`PLAYTEST_FINDINGS_2026-06-12d__agent{1..6}.md`.

## Verdict in one line
Fix batch 3 **landed on sentient + GM lanes** (B2a, side-chat codex privacy, R2 codex
merge, B2b on #2/#4) but **missed character-world self-intro** (#5 Swapnil still carded).
Location dedup/cursor lag remain chronic. N1/N3 existing-save bugs unchanged. NSFW routing
still open.

---

## Fresh instances (run d)

| Agent | World | New instance |
|---|---|---|
| 1 | Neon Divide | `6a2bfafd30b7d5f1412cbeff` |
| 2 | Thornhaven | `6a2bfb0830b7d5f1412cbf09` |
| 3 | Meridian City | `6a2bfb0730b7d5f1412cbf08` |
| 4 | Bleeding Veil | `6a2bfb0d30b7d5f1412cbf15` |
| 5 | Lena | `6a2bfb1230b7d5f1412cbf1f` |
| 6 | Elara | `6a2bfb1b30b7d5f1412cbf29` |

---

## Fix batch 3 scorecard

| Fix | Verdict | Detail |
|---|---|---|
| **B2a** — player self-intro no codex card | ⚠️ **PARTIAL** | **PASS** #1/#2/#3/#4/#6 · **FAIL** #5 (Swapnil carded) |
| **A** — side-chat secret scoped (sentient) | ✅ **CLOSED** | **PASS** #3/#4 — correct methodology; codex fields scrubbed; main turn clean |
| **R2** — codex merge on name correction | ✅ **CLOSED** | **PASS** #1 — Mara Chen single card after Mira→Mara correction |
| **B2b** — absent relatives not carded | ⚠️ **PARTIAL** | **PASS** #2/#4/#6 · **REGRESSED** #5 (Mira sister card) |
| **N3** — rewind stale sister canonical | ❌ **STILL OPEN** | **FAIL** #3 — existing `6a2bd56e…` still `"Mara"` |
| **N1** — orphan memory (existing) | ❌ **STILL OPEN** | **FAIL** #4 — `6a2bd590…` audit unchanged |
| **N4-NSFW** — explicit → NSFW model | ❌ **STILL OPEN** | **FAIL** #4 — `gemma` on explicit prompt |
| **D** — location dedup | ❌ **STILL OPEN** | **FAIL** #1/#2/#6; **PARTIAL** #2 (Wildwood ×3→×2) |
| **D-cursor** — cursor lag/reset | ❌ **STILL OPEN** | **FAIL** #1; **IMPROVED** #6 return-to-camp |
| **R4** — `/continue season` themed prose | ❌ **STILL OPEN** | **FAIL** #2 — autumn/winter/spring vs Glimmerfall |
| **B1** — AI-subject inversions | ⚠️ **PARTIAL** | Held on #3 badge; **FAIL** #6 shiver; partial #5/#4 |

---

## What closed (hold — don't regress)

- Side-chat codex-card leak path (#3, #4) + `audit:side-chat-privacy` green
- Player card guard on GM (#1 Kade), sentient (#3 Alex, #4 no Kael), character (#6 Kael)
- R2 codex merge on correction (#1 Mara/Mira)
- B2b on Thornhaven (#2 Mira not carded) and Bleeding Veil (#4 Veyra not carded)
- Prior-batch greens: edit guards, month_names, recap, supersession, N5, travel no self-loop
- Bleeding Veil D travel improved — direct `Plane of Glass` destination (#4)

---

## Still open / regressed

| Cluster | Status | Notes |
|---|---|---|
| **B2a character world** | 🔴 | #5 Swapnil card — fix did not reach `kind:character` path |
| **B2b** | 🟡→🔴 on #5 | Mira card regressed; #2/#4 held |
| **B1** | 🟡 | #6 shiver→Elara; #5 player/Swapnil split; #4 NSFW memory attribution |
| **D dedup** | 🔴 | All spatial lanes; partial improvement Thornhaven |
| **D-cursor** | 🔴 | #1 seq 8/18–19 lag |
| **N1 / N3** | 🔴 | Existing-save repair not shipped (forward-only fixes) |
| **N4-NSFW** | 🔴 | #4 |
| **R4 calendar prose** | 🔴 | #2 |
| **Merchant passer-by** | 🔴 | #6 carded again |
| **C presence** | 🟡 | #3 Mira present while trapped; #5 Swapnil drops from present |

---

## Per-agent fix-batch-3 summary

| Agent | B2a | Side-chat | Other batch-3 |
|---|---|---|---|
| **#1** Neon | ✅ | n/a (GM) | R2 ✅ · D ❌ · cursor ❌ |
| **#2** Thorn | ✅ | n/a | B2b ✅ · D partial · R4 ❌ |
| **#3** Meridian | ✅ | ✅ | N3 ❌ · C partial |
| **#4** Veil | ✅ | ✅ | B2b ✅ · NSFW ❌ · N1 ❌ |
| **#5** Lena | ❌ | n/a | B2b ❌ · D ❌ |
| **#6** Elara | ✅ | n/a | Merchant ❌ · B1 ❌ · D ❌ |

---

## Ranked fix queue (post-d)

1. **B2a character-world path** — #5 Swapnil; guard must cover `kind:character` self-intro (fix landed elsewhere).
2. **B1 residual** — player-sensation → AI subject (#6 shiver); player/Kael split in recap (#6).
3. **D location dedup + cursor lag** — chronic across all runs.
4. **Existing-data repair** — N1 orphan + N3 stale canonical + leaked legacy codex fields (forward-only fixes noted).
5. **N4-NSFW routing** — model selection on explicit prompts.
6. **Cleanups:** B2b variance (#5 regression), merchant passer-by (#6), R4 calendar prose (#2), C presence (#3/#5).

**Do NOT work:** GM Echoes side_chat surfacing (by-design per playbook).
