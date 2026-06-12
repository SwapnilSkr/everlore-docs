# Agent-3 Playtest Findings — 2026-06-12b

**Lane:** Sentient world (kind:`world`, is_sentient:`true`, is_nsfw_capable:`false`) — modern city-AI.  
**Player speaks AS a persona to the AI (The City).**

**Template ID:** `6a2bd564afc85d8941c370d2` (title: *Meridian City*)  
**Instance ID:** `6a2bd56eafc85d8941c370d4`

**Total turns driven:** 31 (opening + 30 player turns) across two phases (pre-rewind 11 turns, rewind to seq 15, post-rewind 20 turns).  
**Audits run:** `continuity-audit` (8/8 ok), `audit:location`, `audit:movement`, `audit:location-resolution`, `audit:memory-links` — all green.

---

## Regression Checks (verify live)

| Check | Status | Evidence |
|---|---|---|
| **(a) Player-card guard** — introduce player identity (`"My name is Alex, badge L-4472"`) → NO codex/Bonds card minted for the player | **PASS** | `GET /chronicle/relationships/:iid` shows only *The City* (protagonist) and *Mara*. No "Alex" card. |
| **(b) Bonds — AI protagonist card visible with meters** | **PASS** | Bonds shows *The City* with meters (`trust:53`, `affection:50`, `fear:4`, `rivalry:0`). Player is the implicit self side; no self-matching. |
| **(I) Event-edit wrong field now 400** — `PUT /chronicle/event/:id` with `{"narrative":"..."}` | **PASS** | Returns HTTP **400** with body `{"error":"No editable event fields provided. Use ai_response and/or player_input."}`. Correct key `{"ai_response":"..."}` returns 200 + `choices` + `recuration_queued`. |
| **(E) Explicit-correction supersession** — `"my sister's name is Mira, not Mara"` | **PASS** | The correction memory (`6a2bd6bd7a263366b3cfacd8`) has `updates_memory_ids` populated with 4 IDs. The link code materialized forward links. |

---

## Still-Open Focus (expected to still fail — capture evidence)

### [SEV: HIGH] B1 — Player self-fact mis-attributed to the AI character in curated memories
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** Player stated `"My badge is still L-4472. I am a transit engineer."` (seq 16 post-rewind). The resulting curated memory reads:  
  *"The City recalled its badge number, L-4472, a ghostly remnant of a time when identities were clearer; this memory now serves as a fragile anchor in the ever-shifting landscape of existence."*  
  Subjects = `["The City"]`, Objects = `["badge number L-4472"]`.
- **Expected vs got:** The badge L-4472 belongs to the **player persona (Alex)**, not to The City. The memory extractor has no "player persona ≠ protagonist card" concept for sentient worlds, so first-person facts resolve onto the AI protagonist.
- **Evidence:** `GET /chronicle/memories/6a2bd56eafc85d8941c370d4?q=badge` returns the above atom with `subject_entity_ids` pointing to the city entity.
- **Known gap?** Cluster B1 in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — **still open, confirmed live.**

### [SEV: MED] C — On-scene NPCs missing from `present_characters` when the AI dominates
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** At seq 9 (player in the lobby asking about Mara), the narrative says *"She is in the upper spire"* but `present_characters` was empty (`-`). At seq 10 (`/continue day`) Mara suddenly appears in `present`. At seq 21 (weather bureau), Mara is correctly present. At seq 24 (back at Plaza), Mara is absent (correct — she stayed at the bureau). At seq 27 (`/continue day`), `present` suddenly shows `Mara, Maintenance Worker` at the Plaza even though the narrative placed them elsewhere. At seq 28, `The City` finally appears alongside them.
- **Expected vs got:** `present_characters` should reflect **physically co-located** characters. In sentient worlds, the AI protagonist should always be present, and on-scene humans should appear when the narrative places them there. Instead, presence is inconsistent and sometimes empty even when the scene contains multiple people.
- **Evidence:** Frame dumps: seq 9 `present : -`, seq 10 `present : Mara`, seq 27 `present : Mara, Maintenance Worker`, seq 28 `present : Mara, Maintenance Worker, The City`.
- **Known gap?** Cluster C in merged findings — **still open, confirmed live.**

### [SEV: MED] D1 — Location graph fragments (article/variant duplicate nodes)
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** `GET /chronicle/locations/6a2bd56eafc85d8941c370d4` shows multiple duplicate/variant nodes:
  - `"central tower"` (`6a2bd7b27a263366b3cfada8`) and `"the central tower"` (`6a2bd7ad7a263366b3cfada4`) — article variant.
  - `"Sector 4 hub"` (`6a2bd5ec7a263366b3cfabde`) and `"Sector 4 hub"` (`6a2bd5fb7a263366b3cfabf2`) — same name, different parents (one under `the City`, one under `synchronization node`).
  - `"the Plaza"` (`6a2bd6cd7a263366b3cface6`) and `"The Plaza of the Tides"` (`6a2bd6e87a263366b3cfacf8`) — variant name.
- **Expected vs got:** `resolveLocationAnchor` should normalize/strip articles and variants before dedup; same-name places under the same parent should reuse the existing entity.
- **Evidence:** Location list JSON (see `GET /chronicle/locations/6a2bd56eafc85d8941c370d4` output).
- **Known gap?** Cluster D in merged findings — **still open, confirmed live.**

### [SEV: MED] D3 — Location cursor lags one or more turns behind narrated travel
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:**
  - Seq 17: `"I head to the transit hub"` → location stays `the Plaza`.
  - Seq 29: `"I go to my office in the central tower"` → location stays `the Plaza` (seq 29) and `the Plaza` (seq 30) before finally updating to `the lobby` at seq 30.
  - Seq 31: `"I head to the sector 4 maintenance core"` → location stays `the lobby`.
- **Expected vs got:** Location cursor should follow the player's narrated movement on the same turn. The deterministic `movement-signal.ts` backstop should catch "I go to / head to" verbs.
- **Evidence:** Frame dumps showing `location:` field.
- **Known gap?** Cluster D in merged findings — **still open, confirmed live.**

### [SEV: LOW] F2 — Modern almanac month_names null / not Gregorian
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** `GET /chronicle/calendar/6a2bd56eafc85d8941c370d4` → months are correctly populated with Gregorian names (`January` … `December`).
- **Expected vs got:** Expected null (per merged findings) — but actually got correct Gregorian months.
- **Evidence:** Calendar JSON shows `month_names` present.
- **Known gap?** Cluster F2 in merged findings — **NOT reproduced in this instance.** Possible fix or instance-specific variance.

### [SEV: LOW] K — Branch-timeline turn hang after switching active timeline
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** Forked timeline `branch_1781258291658` at seq 15, switched active timeline to it, sent a chat turn (`"This is a branch turn. What do you see, city?"`). Turn completed successfully in ~25s (seq 31).
- **Expected vs got:** Expected 120s timeout/hang (per merged findings). Instead, turn completed normally.
- **Evidence:** `generation_complete` received with event id `6a2bd8527a263366b3cfade9`.
- **Known gap?** Cluster K in merged findings — **NOT reproduced in this instance.** May be fixed or conditionally triggered.

---

## New Bugs / Corruption-Class

### [SEV: MED] Sentient entity (The City) intermittently absent from `present_characters`
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** In a sentient world, the AI protagonist is the conversation partner and should be physically present in every scene. However, `present_characters` was empty (`-`) on many turns: seq 15, 16, 17, 19, 20, 23, 24, 25, 26, 27, 30, 31. The City only appeared sporadically (seq 18, 21, 22, 28, 29).
- **Expected vs got:** The sentient protagonist should ALWAYS be in `present_characters` because the player is literally talking to it.
- **Evidence:** Frame dumps showing `present : -` on turns where the city is clearly narrating and responding.
- **Known gap?** NEW — not explicitly in merged findings. Related to Cluster C but distinct: it's about the AI itself being omitted, not just other NPCs.

### [SEV: MED] Character codex re-minted after rewind with stale name
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** After rewind to seq 15, the character card for the sister was re-minted with name `"Mara"` (old name) even though the pre-rewind player had corrected it to `"Mira"`. The new card (`6a2bd8bfafc85d8941c37114`) has aliases `["Mara","Mira"]` but the canonical name stayed `"Mara"`. The player had explicitly said `"My sister's name is actually Mira, not Mara"` before rewind.
- **Expected vs got:** Rewind should restore the state at the target sequence. At seq 15, the correction had already happened (seq 14), so the character should have been named `"Mira"` or at least carried the correction forward. Instead, the codex reverted to the pre-correction canonical name.
- **Evidence:** `GET /chronicle/relationships/6a2bd56eafc85d8941c370d4` post-rewind shows `"name":"Mara"` with aliases `["Mara","Mira"]`.
- **Known gap?** NEW — rewind may not propagate canonical-name updates from corrected memories into re-minted codex cards.

### [SEV: LOW] Final turn in long batch timed out in harness but committed on server
- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4`
- **Repro:** The 20-turn post-rewind batch (seq 15–34) timed out after 15 minutes on the last step (`"The pumps are still broken. I need to repair them."`). The script exited with code 2. However, the event WAS committed (`seq 32`, event `6a2bdc477a263366b3cfaec2`).
- **Expected vs got:** The harness should wait for `generation_complete` before proceeding. The timeout may be too aggressive for long batches, or the WS frame may have been lost/delayed.
- **Evidence:** Script output: `TIMEOUT — no completion in 120s`. Event count in DB = 32.
- **Known gap?** NEW — harness/UX issue, not a data corruption bug.

---

## §5 Chronicle Endpoint Coverage

All endpoints were exercised and returned data:

| Endpoint | Status | Notes |
|---|---|---|
| `GET /chronicle/recap/:iid` | ✅ | `spine` present, `open_threads` 5 items, `bonds` shows Mara. `when` is null. |
| `GET /chronicle/events/:iid` | ✅ | 32 events returned. |
| `GET /chronicle/memories/:iid` | ✅ | 31 memories. Search `q=badge` works. |
| `GET /chronicle/calendar/:iid` | ✅ | Gregorian months correct. `current_time_anchor` at seq 32. |
| `GET /chronicle/threads/:iid` | ✅ | 11 open threads, 0 resolved. |
| `GET /chronicle/relationships/:iid` | ✅ | 2 characters (The City, Mara). |
| `GET /chronicle/relationships/:iid/:charId/memories` | ✅ | 6 memories for Mara. |
| `GET /chronicle/locations/:iid` | ✅ | 17 places, some fragmented. |
| `GET /chronicle/locations/:iid/:locId` | ✅ | 1 event + 1 memory for sector 4 maintenance core. |
| `GET /chronicle/side-chats/:iid` | ✅ | 1 thread (Mara, 2 turns). |
| `GET /chronicle/side-chats/:iid/:charId` | ✅ | 2 side-chat events for Mara. |
| `PUT /chronicle/event/:id` (wrong key) | ✅ | 400 rejected. |
| `PUT /chronicle/event/:id` (correct key) | ✅ | 200 + recuration + choices returned. |
| `PUT /chronicle/memory/:id` | ✅ | 200 success. |
| `PUT /chronicle/character/:id` | ✅ | 200, aliases updated, appearance set. |
| `POST /chronicle/replay/:id` | ✅ | 200, variant generated with choices. |
| `POST /chronicle/rewind/:iid` | ✅ | 200, deleted 17 events + 15 memories. |
| `POST /chronicle/calendar/:iid/timeline` | ✅ | Forked `branch_1781258291658`. |
| `PUT /chronicle/calendar/:iid/timeline/active` | ✅ | Switched to branch and back to main. |

---

## §5 Audits

| Audit | Result |
|---|---|
| `continuity-audit` (via `GET /admin/instances/:iid/continuity-audit`) | **8/8 ok** — event sequence integrity, single protagonist, codex↔entity linkage, memory/edge refs, summary bounds, time cursor, location cursor all healthy. |
| `bun run audit:location` | ✅ ALL INVARIANTS HELD |
| `bun run audit:movement` | ✅ 45 passed, 0 failed |
| `bun run audit:location-resolution` | ✅ ALL INVARIANTS HELD |
| `bun run audit:memory-links` | ✅ 9 passed, 0 failed |

---

## Summary

- **Regression checks:** All 4 pass (player-card guard, Bonds visibility, event-edit 400, correction supersession).
- **Still-open bugs confirmed live:** B1 (player-fact mis-attribution), C (presence inconsistency), D1/D3 (location fragmentation + cursor lag).
- **Not reproduced:** F2 (Gregorian calendar is actually correct), K (branch turns work).
- **New bugs:** Sentient entity intermittently absent from `present_characters`; codex re-mint after rewind reverts corrected canonical name; harness timeout on long batches.
- **Highest priority:** B1 (silent identity corruption in memories) and D1 (location fragmentation poisons place-recall).

**World left in place:** Template `6a2bd564afc85d8941c370d2` + Instance `6a2bd56eafc85d8941c370d4` (seq 32, main timeline, branch `branch_1781258291658` also exists).
