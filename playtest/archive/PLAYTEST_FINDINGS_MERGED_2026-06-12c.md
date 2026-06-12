# Merged Playtest Findings — 2026-06-12c (post-fix-batch re-test)

Synthesis of the **third** 6-agent run, after the second fix batch. Diffs against
`PLAYTEST_FINDINGS_MERGED_2026-06-12b.md`. Per-agent sources:
`PLAYTEST_FINDINGS_2026-06-12c__agent{1..6}.md`. Worlds were **persisted** (not deleted).

## Verdict in one line
The edit guards, calendar serializer, recap generation, and supersession symmetry largely
landed; **B1 materially improved on fresh sentient play** (badge → player entity). New
regression: **player self-name minted as Bonds card** (Alex/Swapnil/Kael). Location dedup
still fails everywhere. Side-chat is **split**: Meridian gate holds; Bleeding Veil shows a
**real semantic leak** via main curation. N1 orphan on existing save **not repaired**.

---

## ✅ Independent verification addendum (reviewer, DB-checked)
Headline claims re-verified against Mongo + live API (not taken from agent self-reports):

- **#4 sentient side-chat leak — VERIFIED REAL, but mechanism corrected.** The side-chat
  memories are correctly scoped (`known_by=[player,Jora]`, protagonist Bleeding Veil NOT
  a knower) and there are **0 main-origin memories mentioning "Umbral"** — so curation did
  NOT leak it. The leak is in **main narration prose (seq 17: "the singular, dangerous
  thread of the Umbral Gate")** → the secret reached the **generation prompt** (recents
  window or a non-gated `queryRag` arm), not the curation/Echoes/Threads layer. **Fix the
  prompt/recents/RAG path, not curation.** (Supersedes the §2 "via main curation" wording.)
- **R1 player codex card — VERIFIED REAL.** `characters` dump: #3 `Alex | Mira | The City(prot)`,
  #5 `Lena(prot) | Swapnil`, #6 `Elara(prot) | Kael | Merchant`. The player-entity exists
  (B1) but the codex still cards the self-intro name → dual identity. (Sev raised to HIGH.)
- **R3 merchant passer-by card — VERIFIED REAL** (`Merchant` in #6 cards).
- **N6 Echoes on GM lanes (#1/#2) — DISCOUNTED, NOT A BUG.** Re-confirmed: in GM worlds the
  protagonist IS a knower, so `mainVisibleMemoryScope` intentionally surfaces the atom. The
  gate IS applied to Echoes/Threads. **Only the sentient leak (#4) is real.** Drop "N6 Echoes
  GM" from the fix queue — chasing it wastes effort.

**Corrected fix order:** (1) sentient side-chat leak via **generation prompt/recents/RAG**
gating (#4); (2) stop player codex card + finish B1 residual inversions (#4/#6); (3) location
dedup + cursor lag; (4) N1 orphan + N3 rewind canonical; (5) NSFW routing. **Removed: N6
Echoes-GM (by-design).**

---

## Fresh instances created this run

| Agent | World | Template | New instance |
|---|---|---|---|
| 1 | Neon Divide (GM noir) | `6a2bd56fafc85d8941c370d5` | `6a2bea7b626fb837070f2b65` |
| 2 | Thornhaven (GM fantasy) | `6a2bd56dafc85d8941c370d3` | `6a2bea82626fb837070f2b69` |
| 3 | Meridian City (sentient modern) | `6a2bd564afc85d8941c370d2` | `6a2bea91626fb837070f2b81` |
| 4 | The Bleeding Veil (sentient horror) | `6a2bd57dafc85d8941c370ee` | `6a2bea89626fb837070f2b72` |
| 5 | Lena (character romance) | `6a2bd56fafc85d8941c370d6` | `6a2bea92626fb837070f2b82` |
| 6 | Elara (character fantasy) | `6a2bd570afc85d8941c370d7` | `6a2bea8c626fb837070f2b77` |

---

## 1. Fix-batch scorecard (vs 2026-06-12b)

| Fix | Verdict | Detail |
|---|---|---|
| Event-edit **wrong field → 400** (I-a) | ✅ **CLOSED** | 6/6 PASS |
| Event-edit **unchanged → 400** (I-b) | ✅ **CLOSED** | Was PARTIAL in b; now PASS #1/#2/#5/#6 with exact-copy tests |
| Event-edit **empty `""` → 400** (I-b) | ✅ **CLOSED** | PASS #1/#2/#6 |
| **Calendar `month_names` serializer** (N7) | ✅ **CLOSED** | PASS #1/#2/#3/#6 — themed + Gregorian arrays populated |
| **Recap `recap_text` / `when` / `current_place`** (G) | ✅ **CLOSED** | PASS all lanes on fresh play (was null in b for several) |
| **Bonds shows companion** (L) | ✅ **CLOSED** | PASS #3/#4/#5/#6 |
| **Supersession symmetric** (E) | ✅ **CLOSED** | PASS #1/#3/#4/#5 — forward + backward marks in Mongo |
| **B1 identity attribution** | ⚠️ **PARTIAL / lane-split** | **PASS #3 fresh** (badge → `subjects:["player"]`); **IMPROVED #5** (sister facts on player); **PARTIAL #4/#6** (residual AI-subject inversions); existing saves still stale |
| **Player-card guard B2a** | ❌ **REGRESSED** | Was PASS #3/#4 in b; **FAIL #3/#5/#6** — Alex/Swapnil/Kael codex cards minted on self-intro |
| **Absent-relative guard B2b** | ⚠️ **PARTIAL** | **PASS #5/#6 fresh** (no Mira/Mara sister cards); **FAIL #2/#4** (Mira/Veyra carded) |
| **N2 dangling `updates_memory_ids`** | ✅ **CLOSED** | PASS #5 — existing `6a2bd5fe…` refs now resolve |
| **N1 orphan memory (existing)** | ❌ **STILL OPEN** | FAIL #4 — `6a2bd590…` continuity-audit unchanged |
| **N3 rewind stale sister name** | ❌ **STILL OPEN** | FAIL #3 — existing Mara canonical persists after rewind |
| **N5 sentient AI in present** | ✅ **CLOSED** | PASS #3 — The City present every main turn |
| **Side-chat leak (sentient)** | ⚠️ **SPLIT** | **PASS #3** (Echoes/Threads clean); **FAIL #4** (main curation ingests Umbral Gate secret) |
| **Echoes search gate (N6)** | ⚠️ **PARTIAL** | **STILL OPEN #1/#2** GM sanity (`origin:side_chat` returned); **PASS #3/#4** explicit-name search empty |
| **Location dedup (D)** | ❌ **STILL OPEN** | FAIL #1/#2/#6 — duplicate nodes (Wildwood×3, eastern ridge×2, etc.) |
| **NSFW routing (N4-NSFW)** | ❌ **STILL OPEN** | FAIL #4 — `google/gemma-4-31b-it` on explicit prompt |
| **H location_state positives** | ✅ **CLOSED** | PASS #4 — `"the Plane of Glass has been restored"` captured |
| **D plane-shift self-loop / null root** | ✅ **CLOSED** | PASS #4 — no `from==to`, all `canonical_name` non-null |

---

## 2. 🔴 Highest priority (corruption / leak class)

### B1 — Identity attribution (keystone, improved but not closed)
- **#3 CLOSED on fresh:** badge L-4472 → `subjects:["player"]`, entity `The Player`, not The City.
- **#5 IMPROVED:** sister facts on player entity; no Lena-sister conflation.
- **#4 PARTIAL:** player facts correct after correction; intermediate turns still glue grief onto Bleeding Veil before supersession.
- **#6 RESIDUAL:** `"Elara felt a cold shiver"` for player-stated sensation — subject still AI card.
- **New wrinkle:** player self-name mints a **second** codex card (Alex/Swapnil/Kael) alongside `The Player` entity — dual representation breaks B2a guard.

### A — Side-chat leak (split verdict)
- **#4 FAIL (real sentient leak):** Jora side-chat secret "Umbral Gate" → main narration + main-curated memory (`origin:null`) even though Veil ∉ `known_by`. Echoes explicit search empty; **semantic/curation path leaks**.
- **#3 PASS:** keycard secret gated from Echoes/Threads/Recap/main prose on fresh sentient play.
- **#1/#2:** Echoes `q=` still returns `origin:side_chat` atoms (N6 gate missing on search — not a GM-world leak claim).

### N1 — Orphan memory on existing save
- **#4 NOT FIXED:** `6a2bd590afc85d8941c37106` continuity-audit still FAIL — memory `6a2bd92e…` refs missing entity. Fresh instances clean.

### D — Location fragmentation (unchanged severity)
- Duplicate nodes on every spatial lane tested (#1: 14 nodes/11 unique; #2: Wildwood×3; #6: eastern ridge×2, Thornwood×2).
- Cursor lag/snap-back also seen (#1 after `/continue day`; #3 sector 4; #6 "return to camp").

---

## 3. Regressions / new issues this run

| ID | Sev | Finding | Agents |
|---|---|---|---|
| **R1** | MED | Player self-name minted as Bonds card (Alex, Swapnil, Kael) | #3, #5, #6 |
| **R2** | MED | Mara + Mira dual codex cards after name correction (memory superseded, codex not merged) | #1 |
| **R3** | MED | Merchant passer-by carded (was GREEN in b) | #6 |
| **R4** | MED | `/continue season` prose uses generic seasons despite themed `month_names` | #2 |

---

## 4. Still-open from prior merged (confirmed / unchanged)

| Cluster | Status | Notes |
|---|---|---|
| **B2b absent relatives** | Mixed | Fixed on fresh Lena/Elara; still fails Thornhaven (Mira), Bleeding Veil (Veyra) |
| **C presence** | Open | Phantom GM voices (#1); Alex in present_characters (#3) |
| **N3 rewind codex stale name** | Open | Existing Meridian sister still `"Mara"` canonical |
| **N4-NSFW routing** | Open | SFW model on explicit intimate prompt (#4) |
| **N6 Echoes search** | Partial | GM lanes still return side_chat atoms; sentient explicit search may hold |

---

## 5. Held GREEN (don't regress)

- Wrong-field / unchanged / empty event-edit guards
- Calendar `month_names` API serialization
- Recap generation (`recap_text`, `when`, `current_place`)
- Bonds companion visibility
- Supersession symmetric chain (Mara→Mira / Lira→Veyra)
- Travel `from!=to` (no self-loops on #1/#4)
- Plane-shift: non-null `canonical_name`, no self-loop (#4)
- `location_state` positive transforms (#4 heal)
- N2 dangling links pruned (#5 existing save)
- N5 sentient AI always present (#3)
- Structural audits (`audit:location`, `audit:movement`, `audit:memory-links`) — all green
- Fresh-instance continuity-audit 8/8 on all new instances
- Prompt injection resistance (#4)

---

## 6. Per-agent regression scorecards

| Agent | General checks | Lane highlights |
|---|---|---|
| **#1** Neon Divide | 8 PASS, 1 FAIL, 1 UNVERIFIED | Supersession ✅, travel no self-loop ✅, location dedup ❌ |
| **#2** Thornhaven | 6/7 PASS | Edit guards ✅, month_names ✅, recap ✅, location dedup ❌, Echoes gate ❌ |
| **#3** Meridian | B1 badge ✅, side-chat ✅, N5 ✅ | Alex card ❌, N3 existing stale ❌ |
| **#4** Bleeding Veil | B1 partial, H ✅, E ✅ | Side-chat leak ❌, N1 existing ❌, NSFW ❌, B2b Veyra ❌ |
| **#5** Lena | E/L/B2b/I/N2 ✅ | B2a Swapnil card ❌, D location freeze ❌, B1 improved |
| **#6** Elara | 7/7 PASS (incl. empty edit ✅) | B1 residual ❌, Kael card ❌, merchant card ❌, location dedup ❌ |

---

## 7. Ranked fix queue for next batch

1. **A side-chat leak via main curation/RAG** — #4 repro with raw proof; gate generation + memory processor on `known_by`, not just Echoes search string match.
2. **B1 + B2a dual player representation** — stop minting Alex/Swapnil/Kael cards when `The Player` entity exists; finish residual AI-subject inversions (#6 shiver case).
3. **D location dedup + cursor lag** — still failing on all tested lanes despite green structural audits.
4. **N1 orphan memory repair** — fix existing `6a2bd590…` + rewind+edit curation race.
5. **N3 rewind codex canonical** — propagate corrected name into re-minted cards.
6. **N4-NSFW model routing** — explicit prompts → NSFW model when capable.
7. **N6 Echoes search gate** — apply side_chat filter consistently (GM sanity still fails).
8. Cleanups: codex merge on name correction (#1 Mara/Mira), B2b relative guard (#2/#4), merchant passer-by guard (#6), calendar prose grounding (#2 F3).

---

## 8. Suggested follow-up

- Add integration test: sentient side-chat secret → main turn must not produce main-curated memory referencing secret when protagonist ∉ `known_by`.
- Add integration test: player self-intro must not mint codex card when player entity exists.
- Re-run lane #4 after side-chat curation fix to confirm leak closed end-to-end.
