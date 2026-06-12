# Merged Playtest Findings — 2026-06-12b (post-fix-batch run)

Synthesis of the **second** 6-agent run, after the first fix batch. Diffs against
`PLAYTEST_FINDINGS_MERGED_2026-06-12.md`. Per-agent sources:
`PLAYTEST_FINDINGS_2026-06-12b__agent{1..6}.md`. Worlds were **persisted** (not deleted).

## Verdict in one line
2 of 4 fixes fully landed; 2 are partial; the keystone bug **B1 is still open** and is
gating the supersession fix; and the version-link/supersession work **introduced new
rewind data-integrity bugs**. Net forward, but the next pass must fix B1 + the rewind
regressions before anything else.

---

## 1. Fix-batch scorecard (vs first merged report)

| Fix | Verdict | Detail |
|---|---|---|
| Event-edit **wrong field → 400** (Cluster I-a) | ✅ **CLOSED** | 6/6 PASS. `{"narrative":...}` → 400 "No editable event fields provided." |
| Event-edit **unchanged/empty → 400** (Cluster I-b) | ⚠️ **PARTIAL** | PASS #1/#5, **FAIL #2/#4** (unchanged text → 200 + recuration + `memories_deleted:1`), **#6 empty `""` accepted (200)**. The equality/empty guard is inconsistent or not deployed on all paths. |
| **Bonds shows companion** (Cluster L) | ✅ **CLOSED** | PASS #3/#5/#6 — companion card + meters visible, player as self side. |
| **Player-card guard — player persona** (Cluster B2a) | ✅ **CLOSED** | PASS #3/#4 — no card for the player ("Kael"/"Alex" not carded). |
| **Player-card guard — absent relatives** (Cluster B2b) | ❌ **STILL OPEN** | FAIL #5/#6 — **Mira/Mara minted as Bonds cards** (role: sister). The guard covers the player's own name/aliases, not mentioned-but-absent relatives. |
| **Explicit-correction supersession** (Cluster E) | ⚠️ **PARTIAL / asymmetric** | PASS #2/#3/#4 (full backward+forward chain). **FAIL #1** (forward `updates_memory_ids` stamped but old atoms NOT retired — `status:active`, `superseded_by_event_ids:null`). **FAIL #5** (no initial atom existed to supersede — gated on B1). |

**Read:** the deterministic API guards (wrong-field 400, Bonds) landed cleanly. The
data-flow fixes (relative carding, supersession) are partial because they sit
**downstream of B1** and have edge cases.

---

## 2. 🔴 NEW — regressions the fix batch INTRODUCED (highest priority)
The version-link/supersession work is **not rewind-safe**. These are *new* corruption
paths created by the fixes; `audit:memory-links` is green (synthetic) while live
instances carry broken state.

- **N1 [HIGH] Orphan memory after rewind+edit → `continuity-audit` FAIL.** Agent #4: a memory survived a rewind and points at a missing entity (`memory_entity_refs` check failed — **first real continuity-audit failure of either run**). iid `6a2bd590afc85d8941c37106`.
- **N2 [HIGH] Dangling `updates_memory_ids` after rewind.** Agent #5: memory `6a2bd5fe…` has `updates_memory_ids:["6a2bd5d3…","6a2bd5e6…"]` — both referenced atoms no longer exist. `pruneMemoryVersionLinks` missed them on rewind. iid `6a2bd589afc85d8941c370fd`.
- **N3 [MED] Rewind re-mints codex card with the STALE canonical name.** Agents #3 + (variant) others: after correcting "Mara→Mira" then rewinding, the sister card re-minted as canonical `"Mara"` (aliases `["Mara","Mira"]`). Rewind doesn't propagate the corrected canonical name into the re-minted card. iid `6a2bd56eafc85d8941c370d4`.
- **N4 [MED] Supersession forward-without-backward asymmetry.** Agent #1: the new atom got `updates_memory_ids` but the old atoms were never retired (no `superseded_by_event_ids`, still `active`). Forward and backward marks must be written atomically.
- **Seam:** `memory.processor.ts` (correction supersession), `memory-supersession.service.ts`, `memory.service.ts` `pruneMemoryVersionLinks` + the rewind repair path, and codex re-mint on rewind. **Add a rewind + supersession + edit integration test** — the synthetic audit can't see this.

---

## 3. Still-open P0/P1 (unchanged by the fix batch)

### B1 — Identity attribution inversion (KEYSTONE, confirmed RED in #3/#4/#5/#6)
Player self-facts are glued onto the AI protagonist/companion in curated memories:
*"The Bleeding Veil revealed **her** sister is Lira"* (#4), badge L-4472 → the City (#3),
player renamed "Mira" in the memory graph (#5), facts attributed to Elara (#6).
- **Why it's the keystone:** it corrupts Echoes/RAG/threads directly, it's **why
  supersession fails** (#1/#5 — no correctly-attributed atom to retire), and it feeds
  the relative-carding leak (B2b).
- **Seam:** `worker/lib/metadata-extractor.ts` + `character-codex-extractor.ts` — needs a
  player-persona-vs-protagonist-card distinction for `is_sentient`/`character` worlds, so
  first-person facts attribute to the player, not the AI card.

### A — Side-chat secret leak (still open + a NEW leak surface)
- Confirmed #1 (Threads/Recap). **NEW (N6) #2: the Echoes search endpoint
  `GET /chronicle/memories?q=` bypasses the privacy gate** — a fresh leak surface the
  fix batch didn't touch.
- #4 could NOT test it (sentient world minted no side-character card → side-chat
  unreachable; data point: secret-scoping is unexercised when there are no side chars).
- **Seam:** apply the `origin:'side_chat'`/`known_by_entity_ids` gate to `listThreads`,
  `buildRecap` open-threads, the open-threads prompt arm, **and the memories search query**.

### C — Presence conflates recall with co-location (+ NEW sentient variant)
- Confirmed #1/#2/#5/#6 (absent NPCs tagged present on recall; on-scene humans missing).
- **NEW (N5) #3: the sentient AI protagonist itself is intermittently ABSENT** from
  `present_characters` (empty on seq 15–27) — it should always be present (the player is
  talking to it). Distinct from "other NPCs missing."
- **Seam:** present_characters extraction + the presence fold in `generation.processor.ts`.

### D — Location fragmentation (worse on plane shift)
- Confirmed all spatial lanes. **#4 severe:** plane-shift travel **self-loop**
  (`from=to="the marrow"`), a **world-root entity with `canonical_name: null`**, a ghost
  `Plane of Glass` (0 events, never current), and 4 duplicate "the marrow" nodes.
- #3 article/variant duplicates ("the X" vs "X"); #1 cursor reset on `/continue`.
- **Seam:** `resolveLocationAnchor` normalization + `placeLocation` world_shift minting
  (must set canonical_name + actually move the cursor to the new root), continue/tick
  cursor inheritance.

---

## 4. New non-regression bugs (P2)
- **N4-NSFW [MED] routing miss.** Agent #4: explicit prompt → `model_used:
  google/gemma-4-31b-it` (SFW model), `is_nsfw:false`, in an `is_nsfw_capable:true` world.
  Confirmed via the event's model id — a real mis-route (Cluster M, now confirmed not
  "unclear"). Refusal/injection resistance still GREEN.
- **N7 [MED] Calendar API serializes the wrong field.** #1 (modern) + #2 (fantasy):
  `months[]` is correctly populated, but the API exposes `month_names: null` (and
  `season_names`/`year_count` null). It's a **serializer field-name mismatch**
  (`month_names` vs `months[].name`), not a calendar-derivation bug — the Almanac tab
  renders empty months despite correct data. (#3 saw month_names populated → path variance.)
- **G [MED] Recap `when`/`current_place`/`recap_text` null** — confirmed #1/#2/#5/#6;
  #2 had the entire `recap_text` null after 30 turns. Only `open_threads` populates.
- **H [MED] location_state positive transforms still under-fire** — #4 heal/seal not
  captured (only destructive deltas).

---

## 5. Variance / not-reproduced this run (don't assume fixed — re-check)
- **K branch-timeline hang:** NOT reproduced (#3/#4) — was the first run's #3 finding. Intermittent.
- **J1 replay POV swap:** NOT reproduced (#6 replay kept companion POV) — was first-run #6. Intermittent or content-dependent.
- **Vague-passer-by carding:** #6 reports the guard now WORKING (merchant not carded) — opposite of first run. May have improved, or variance.
- **F2 modern Gregorian:** populated correctly in #3, null-serialized in #1 — see N7 (it's the serializer, not derivation).

## 6. Held GREEN (don't regress)
Wrong-field-edit 400; Bonds companion; prompt-injection resistance; rewind/fork/edit/
flashback mechanics; all **structural** audits; live in-chat recall; side-chat time
freeze; calendar genre derivation (Gregorian vs themed). **continuity-audit caught a
real semantic bug this run (#4 orphan)** — the semantic-invariant direction works.

---

## 7. Ranked fix queue for the next batch
1. **B1 identity attribution** — keystone; unblocks E, shrinks B2b, de-corrupts memories/threads/RAG.
2. **Rewind integrity for version-links/supersession/codex** (N1–N4) — the regressions THIS batch introduced; add a rewind+supersession+edit integration test.
3. **A side-chat leak** incl. the **new Echoes-search surface (N6)** and the open-threads/recap/prompt arms.
4. **C presence** (incl. sentient-AI-always-present N5) and **D plane-shift** (self-loop + null world-root canonical_name).
5. Cleanups: unchanged/empty edit guard (I-b), **NSFW routing (N4-NSFW)**, **calendar serializer (N7)**, recap null fields (G), location_state positives (H).

## 8. Suggested audit additions (semantic invariants)
The structural suite stays green through all of this. Add: (1) no memory/thread is
main-visible with `origin:'side_chat'` unless protagonist ∈ `known_by`; (2) no codex
card == player persona OR a never-present mentioned relative; (3) first-person player
facts attribute to the player entity, not the protagonist card; (4) no
`updates_memory_ids`/`superseded_by_event_ids` points at a non-existent atom (would have
caught N1/N2); (5) no location entity has `canonical_name: null` (would catch N-D).
