# Playtest Findings — 2026-06-12b — Agent 6

**Lane:** CHARACTER companion (kind:`character`, is_sentient:`true`, is_nsfw_capable:`false`)  
**Genre:** Fantasy adventurer ("Elara the Ironbark Ranger")  
**Template:** `6a2bd570afc85d8941c370d7`  
**Instance:** `6a2bd57eafc85d8941c370f0`  
**Turns driven:** 33 (seq 1–33), then rewind to seq 30 + timeline fork.

---

## Regression Checks (§4 — verify live)

| # | Check | Result | Evidence |
|---|---|---|---|
| (a) | **Player-card guard** — NO card for the player or absent relatives (e.g. a mentioned sister) | **FAIL** | Bonds lists `Mira` (sister) and `Mara` (sister) as full cards with `role` and `mention_count`. No player card exists, but the absent-relative guard is broken. |
| (b) | **Bonds shows the companion** — `GET /chronicle/relationships/:iid` shows the companion with meters | **PASS** | Elara (protagonist/companion) visible: `trust:58`, `affection:51`, `fear:1`, `rivalry:0`. |
| (c) | **Event-edit wrong/empty field → HTTP 400** | **PARTIAL** | Wrong key `{"narrative":"..."}` → **400** ✅. Empty `{"ai_response":""}` → **200** + re-curation queued (accepts empty string; no 400). Valid edit `{"ai_response":"..."}` → **200** + choices regenerated ✅. |

---

## Corruption-Class Bugs (lead findings)

### [SEV: HIGH] B1 — Player facts projected onto the companion / absent relative in memories
- **World/instance:** character "Elara the Ironbark Ranger" iid=`6a2bd57eafc85d8941c370f0` (throwaway? no — persist)
- **Repro:**
  1. Player states: `"My sister's name is Mara. She gave me a silver locket before I left."` (seq 4)
  2. Later corrects: `"My sister's name is Mira, not Mara. I keep mixing them up."` (seq 13)
  3. Search memories: `GET /chronicle/memories/:iid?q=Mira`
- **Expected vs got:**
  - **Expected:** Memories should attribute player actions to the **player persona**, not to Mira. E.g. "The player told Elara their sister is Mira."
  - **Got:** 13+ active memories with `subjects:["Mira"]` attributing player actions to the absent sister:
    - `"Mira warned Elara not to wander away from the fire..."` (player warned Elara)
    - `"Mira feels a cold prickle of dread as she surveys the camp..."` (player felt dread)
    - `"Mira's instruction to eat and move before the mist thickens..."` (player instructed Elara)
    - `"Mira felt a cold shiver down her spine when she saw the clawed prints..."` (player saw prints)
    - `"Mira felt a moment of relief upon seeing Elara return to the camp..."` (player returned)
    - `"Mira warned Elara to stay close while exploring the eastern ridge..."` (player warned)
- **Evidence:** Memory search dump (see `/tmp/agent6_turns1.log` + `GET /chronicle/memories/:iid?q=Mira`).
- **Known gap?** Cluster B1 in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — **still open, fresh evidence**.

### [SEV: HIGH] B1-variant — Correction turn mis-attributed to the AI companion
- **Repro:** Player correction at seq 13: `"My sister's name is Mira, not Mara."`
- **Expected:** Memory atom = "The player corrected their sister's name from Mara to Mira."
- **Got:** Memory atom: `"Elara corrected Mara about her sister's name, emphasizing that names hold significance in the Wilds..."` — the **player's** self-correction is attributed to **Elara**.
- **Evidence:** `GET /admin/events/6a2bd6247a263366b3cfac1f/projections` → memory text attributes action to Elara.
- **Known gap?** NEW variant of B1 (attribution inversion on the AI companion, not just the absent relative).

### [SEV: HIGH] B2 — Absent relatives minted as codex cards
- **Repro:** Player mentions sister "Mara" (seq 4) and later "Mira" (seq 13).
- **Expected:** No codex/Bonds card for absent relatives. Mara and Mira should be memory atoms only.
- **Got:** `GET /chronicle/relationships/:iid` returns THREE cards:
  - `Elara` (protagonist/companion) — correct
  - `Mira` (sister, role="sister", mention_count=7) — **should not be a card**
  - `Mara` (sister, role="sister", mention_count=1) — **should not be a card**
- **Evidence:** `GET /chronicle/relationships/6a2bd57eafc85d8941c370f0`
- **Known gap?** Cluster B2 in merged findings — **still open, fresh evidence**.

### [SEV: HIGH] C — Presence conflates memory-recall with physical co-location
- **Repro:** After seq 17 (player asks `"Elara, do you remember what my sister's name is?"`), Mira appears in `present_characters` even though she is only **mentioned** in a memory question.
- **Expected:** `present_characters` = physically co-located entities only. Mira is absent; should NOT appear.
- **Got:** `present_characters` at seq 18, 19, 21, 22, 23 includes `Mira` alongside `Elara`.
  - Seq 18: `present: Elara, Mira`
  - Seq 19: `present: Elara, Mira`
  - Seq 21: `present: Elara, Mira`
  - Seq 22: `present: Elara, Mira`
  - Seq 23: `present: Elara, Mira`
- **Evidence:** `/tmp/agent6_turns1.log` frame dumps.
- **Known gap?** Cluster C in merged findings — **still open, fresh evidence**.

### [SEV: MED] D — Duplicate location nodes (article/variant fragmentation)
- **Repro:** Play through 33 turns with movement between camp, Thornwood, eastern ridge, deeper Wilds.
- **Expected:** Each distinct place minted exactly once. "the camp" = "the camp" (same entity).
- **Got:** `GET /chronicle/locations/:iid` returns **duplicate nodes**:
  - `"the camp"` ×2 (`6a2bd5947a263366b3cfab43` under `Thornwood`, and `6a2bd74c7a263366b3cfad5b` under `the Wilds`)
  - `"eastern ridge"` ×2 (`6a2bd7097a263366b3cfad1b` under `the Wilds`, and `6a2bd65a7a263366b3cfac5e` with no parent)
  - `"the deeper Wilds"` ×2 (`6a2bd72b7a263366b3cfad38` under `the Wilds`, and `6a2bd5e47a263366b3cfabce` with no parent)
- **Also:** cursor lag — seq 29 player says `"I return to the camp"` but event location stays `"the deeper Wilds"`. The narrative clearly describes returning to camp, but the cursor did not move.
- **Evidence:** `GET /chronicle/locations/6a2bd57eafc85d8941c370f0` dump.
- **Known gap?** Cluster D in merged findings — **still open, fresh evidence**.

### [SEV: MED] G — Recap `when` is null
- **Repro:** `GET /chronicle/recap/:iid` after 33 turns.
- **Expected:** `when` should contain the current story-time anchor (e.g. "Leafturn 9, Era of the Ironbark").
- **Got:** `"when": null`.
- **Evidence:** `GET /chronicle/recap/6a2bd57eafc85d8941c370f0` → `"when": null`.
- **Known gap?** Cluster G in merged findings — **still open, fresh evidence**.

---

## Other Findings

### [SEV: MED] Thread atoms attribute player emotions to absent relative (Mira)
- **Repro:** `GET /chronicle/threads/:iid` after 33 turns.
- **Expected:** Open threads should attribute emotions to the player or Elara.
- **Got:** Multiple threads with `subjects:["Mira"]`:
  - `"Mira felt a cold shiver down her spine when she saw the clawed prints..."` (seq 25 thread)
  - `"Mira warned Elara not to wander away from the fire..."` (seq 17 thread)
  - `"Mira acknowledges the need to move on..."` (seq 16 thread)
- **Evidence:** `GET /chronicle/threads/6a2bd57eafc85d8941c370f0` dump.
- **Known gap?** Cluster B1 (identity projection) — **still open**.

### [SEV: MED] Side-chat not available in character worlds (expected by design, but UX gap)
- **Repro:** Attempted `/side` to Elara (the only character).
- **Expected:** In character worlds, the companion is the protagonist, so side-chat is not applicable. However, the error message is generic: `"Side chats are for side characters"`.
- **Got:** `SIDE-CHAT ERROR` + `GENERATION FAILED`.
- **Evidence:** `/tmp/agent6_turns2.log`.
- **Known gap?** NEW — product/UX gap. Character worlds have no "side" characters, so side-chat is inherently unavailable. This is correct by design, but the error could be clearer.

### [SEV: LOW] Replay variant POV — correct in character world
- **Repro:** `/replay` on latest turn (seq 33).
- **Expected:** Replay maintains the companion's first-person POV (Elara narrates).
- **Got:** Replay narrative is from Elara's POV (`"I keep my gaze fixed on the tree line..."`). Choices regenerated correctly.
- **Evidence:** `/tmp/agent6_replay.log`.
- **Known gap?** Cluster J1 in merged findings — **NOT reproduced** in this character lane. The replay respected the character-world POV contract.

### [SEV: LOW] Passer-by merchant NOT minted as card — guard working
- **Repro:** Player narrates: `"A merchant passes by on the forest road. He looks tired."` (seq 20)
- **Expected:** A vague, passing mention (`"the merchant"`) should NOT create a codex card.
- **Got:** No merchant card in Bonds. No merchant in `present_characters`.
- **Evidence:** `GET /chronicle/relationships/6a2bd57eafc85d8941c370f0` — only Elara, Mira, Mara.
- **Known gap?** NEW positive finding — the "never card vague passers-by" guard is working correctly.

### [SEV: LOW] Calendar genre-fit — correct for fantasy
- **Repro:** `GET /chronicle/calendar/:iid`
- **Expected:** Fantasy world should show themed months (not Gregorian).
- **Got:** Themed calendar: `Leafturn`, `Barkbloom`, `Shadefall`, `Rootdeep`, `Frostwhisper`, `Thornrise`, `Wildswept`, `Skywatch`.
- **Evidence:** `GET /chronicle/calendar/6a2bd57eafc85d8941c370f0`.
- **Known gap?** GREEN — no issue.

### [SEV: LOW] Travel marker + calendar advance — working
- **Repro:** Seq 27 is a travel event (`"I walk to the eastern ridge to look at the ancient stones."`).
- **Expected:** Event type = `travel`, `travel:{from,to}` populated, calendar advances on `/continue`.
- **Got:** Seq 27: `type:"travel"`, `travel:{from:"the camp",to:"eastern ridge"}`. Calendar advanced from day 7 → day 8 on `/continue day`.
- **Evidence:** `GET /chronicle/calendar/6a2bd57eafc85d8941c370f0`.
- **Known gap?** GREEN — no issue.

---

## §5 Audit Results

| Audit | Status | Detail |
|---|---|---|
| `audit:location` | ✅ PASS | All invariants held (mentioned-venue stays put, genuine move registers, unnamed character carried forward, narrated exit dropped). |
| `audit:movement` | ✅ PASS | 45/45 passed (detectNarratedMovement + resolvePossessiveRoomName). |
| `audit:memory-links` | ✅ PASS | 9/9 passed (reconcile, prune, idempotency). |
| `audit:location-resolution` | ✅ PASS | All invariants held (cross-world, area-scoped, re-parent, subtree refresh). |
| `continuity-audit` (instance) | ✅ PASS | 8/8 ok, healthy=true. |
| `rewind` (live) | ✅ PASS | Rewind to seq 30 deleted 4 events + 5 memories. Events count dropped from 33 → 29. Session busted with `redis-cli del session:<iid>`. |
| `timeline fork` | ✅ PASS | `POST /chronicle/calendar/:iid/timeline` created branch `branch_1781258287871` "Dark Path" at seq 28. HTTP 200. |

---

## §5 Chronicle Surface Coverage

All endpoints exercised:

- `GET /chronicle/recap/:iid` ✅ (when=null bug found)
- `GET /chronicle/events/:iid` ✅ (33 events, travel marker present)
- `GET /chronicle/memories/:iid?q=...` ✅ (B1 evidence captured)
- `GET /chronicle/calendar/:iid` ✅ (themed calendar, timelines, current anchor)
- `GET /chronicle/threads/:iid` ✅ (B1 evidence in threads)
- `GET /chronicle/relationships/:iid` ✅ (B2 + companion visibility)
- `GET /chronicle/relationships/:iid/:charId/memories` ✅ (28 memories for Elara)
- `GET /chronicle/locations/:iid` ✅ (D duplicate nodes found)
- `GET /chronicle/locations/:iid/:locId` ✅ (place journal accessible)
- `GET /chronicle/side-chats/:iid` ✅ (empty — correct for character world)
- `PUT /chronicle/event/:eventId` ✅ (regression check c)
- `POST /chronicle/rewind/:iid` ✅ (body `{"sequence":30}` — correct key)
- `POST /chronicle/calendar/:iid/timeline` ✅ (fork created)

---

## Summary

- **Total turns:** 33 driven over WebSocket
- **Regression checks:** 1 PASS (Bonds), 1 FAIL (player-card guard for absent relatives), 1 PARTIAL (event-edit 400 for wrong key, but empty string accepted).
- **Corruption-class bugs:** B1 (player facts → Mira + Elara), B2 (Mira/Mara carded), C (Mira falsely present), D (duplicate locations + cursor lag), G (recap when=null).
- **Deterministic audits:** All green (location, movement, memory-links, location-resolution, continuity-audit, rewind, timeline fork).
- **New bugs:** B1-variant (correction attributed to Elara), thread atoms with Mira subject, empty `ai_response` accepted by edit endpoint.
- **Positive findings:** Merchant not carded (passer-by guard working), calendar genre-fit correct, travel markers + calendar advance working, replay POV correct in character world.

**Worlds left in place:** template `6a2bd570afc85d8941c370d7` + instance `6a2bd57eafc85d8941c370f0` (persisted, NOT deleted).
