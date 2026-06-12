# Playtest Findings — 2026-06-12b — Agent 2

**QA Agent:** #2  
**Lane:** GM world, high fantasy  
**Template ID:** `6a2bd56dafc85d8941c370d3` ("Thornhaven")  
**Instance ID:** `6a2bd57cafc85d8941c370ea`  
**World kept:** YES (per lifecycle policy)

---

## Regression Checks (§4)

| Check | Result | Notes |
|---|---|---|
| **Event-edit wrong field → HTTP 400** | **PASS** | `PUT /chronicle/event/:id {"narrative":"x"}` returned 400 `"No editable event fields provided. Use ai_response and/or player_input."` |
| **Event-edit unchanged ai_response → HTTP 400** | **FAIL** | `PUT /chronicle/event/:id {"ai_response":"<unchanged text>"}` returned **200** + recuration instead of rejecting. **REGRESSION** — the unchanged-guard fix (Cluster I) did not land live. |
| **Event-edit changed ai_response → HTTP 200 + recurate** | **PASS** | Returned 200 with `recuration_queued:true`, new `choices` and `present_characters` regenerated. |
| **Bonds shows companion in sentient/character worlds** | N/A | GM world lane — not applicable. |
| **Player-card guard in sentient worlds** | N/A | GM world lane — not applicable. |
| **Explicit-correction supersession** | **PASS** | Live "Mira, not Mara" correction: old Mara atoms (`6a2bd5d2…`, `6a2bd64a…`) got `status:superseded` + `superseded_by_event_ids` stamped; correction atom got `updates_memory_ids` linked. Full chain verified in MongoDB. |

---

## Corruption-Class Bugs (P0 / P1)

### [SEV: HIGH] Cluster B — Identity boundary collapse: protagonist minted as NPC card + absent-relative carding
- **World/instance:** GM world "Thornhaven" iid=`6a2bd57cafc85d8941c370ea` (throwaway? no)
- **Repro:** Play 10+ turns in a GM world where the narrator refers to the protagonist with descriptive epithets ("the sentinel", "the traveler", "the rider"). Then `GET /chronicle/relationships/:iid`.
- **Expected vs got:** Expected: the protagonist (Seraphine) is the ONLY player persona; no codex card for the player. Got: **"Sentinel"** minted as a standalone NPC character card (`6a2bdb05afc85d8941c37122`) in the Bonds ledger. Also, the absent sister **"Mara"** was carded as a separate NPC (`6a2bdb05afc85d8941c37125`), and after the correction "Mira, not Mara", a **second** sister card **"Mira"** was minted (`6a2bdb05afc85d8941c37129`). The protagonist's true name (Seraphine) was also curated into a memory atom, but the codex card for the protagonist was not properly linked — instead a "Sentinel" card appeared.
- **Evidence:**
  ```json
  // Bonds/Relationships GET response (after rewind):
  { "name": "Sentinel", "id": "6a2bdb05afc85d8941c37122" }
  { "name": "Mara", "id": "6a2bdb05afc85d8941c37125" }
  { "name": "Mira", "id": "6a2bdb05afc85d8941c37129" }
  ```
  The protagonist's persona is being fragmented into a "Sentinel" card and the absent relative is being carded as a separate NPC instead of being treated as a memory-only entity.
- **Known gap?** Cluster B2 (player/relatives carded in GM worlds). The fix for sentient/character worlds was merged, but GM worlds are ALSO affected when the narrator uses epithets that the extractor doesn't map back to the protagonist name.

### [SEV: HIGH] Cluster B — Duplicate character cards from epithet/variant naming
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** After 15+ turns, `GET /chronicle/relationships/:iid`
- **Expected vs got:** Expected: one card for the hooded figure. Got: **"Stranger"** (`6a2bdb05afc85d8941c37127`) AND **"the stranger"** (`6a2bdb05afc85d8941c37128`) as two separate cards in the Bonds ledger.
- **Evidence:**
  ```json
  { "name": "Stranger", "id": "6a2bdb05afc85d8941c37127" }
  { "name": "the stranger", "id": "6a2bdb05afc85d8941c37128" }
  ```
  Same character, two cards because the prose used capitalized and lowercase variants on different turns. The `codex-dedup-audit` hardening (commit `9101349`) did not prevent this in live play.
- **Known gap?** Cluster B (identity boundary collapse). The kin-epithet fix handled "sister" vs "Twin Sister", but lowercase/uppercase variant dedup ("Stranger" vs "the stranger") is still failing.

### [SEV: HIGH] Cluster A — Side-chat secret memory visible in main Echoes search
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** `/side 6a2bd6a87a263366b3cfacc2 "I know something about the raven you don't. The scroll was written in fae ink."` → wait for completion → `GET /chronicle/memories/:iid?q=fae%20ink`
- **Expected vs got:** Expected: the main memory search should NOT surface the private side-chat secret. Got: the search returned the side-chat atom with `origin: "side_chat"` and `known_by_entity_ids`.
- **Evidence:**
  ```json
  {
    "text": "Stranger acknowledged Traveler's insight about the raven's scroll being written in fae ink...",
    "origin": "side_chat",
    "known_by": ["6a2bd5987a263366b3cfab49", "6a2bd6a87a263366b3cfacc3", "6a2bd60d7a263366b3cfac0a"]
  }
  ```
  The atom is tagged correctly, but it appears in a **main** search query. If the RAG pipeline also fails to filter it, the secret would leak into main narration. The Threads and Recap surfaces did NOT show the secret text in this run, but the Echoes search does — indicating the privacy gate is inconsistently applied.
- **Known gap?** Cluster A — side-chat secret leak into main projections. The fix batch claimed to gate all Lore Tome tabs, but the **Echoes search endpoint** (`/chronicle/memories?q=`) bypasses the gate. This is a new leak surface.

### [SEV: MED] Cluster I — Unchanged ai_response edit silently accepted (regression)
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** `PUT /chronicle/event/:id {"ai_response":"<exact same text>"}`
- **Expected vs got:** Expected: HTTP 400, no-op, no recuration. Got: HTTP 200, `recuration_queued:true`, `memories_deleted:1`, new choices regenerated.
- **Evidence:**
  ```
  HTTP/1.1 200 OK
  {"success":true,"memories_deleted":1,"recuration_queued":true, ...}
  ```
  The edit wasted compute, deleted memories, and regenerated choices for a no-op text change. The fix that was supposed to add an unchanged-guard (Cluster I) is either not deployed or not matching the live text.
- **Known gap?** Cluster I — regression from the June 12 fix batch. `typecheck` + audit may pass, but the live API doesn't reject unchanged edits.

### [SEV: MED] Cluster F — Calendar month_names null in API response
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** `GET /chronicle/calendar/:iid`
- **Expected vs got:** Expected: `month_names` array with the 8 themed months (Bloomrise, Faeweave, Gloomshade, Thornfall, Wyrmrest, Frostbloom, Sunblight, Nightveil). Got: `month_names: null`, `season_names: null`, `year_count: null`.
- **Evidence:**
  ```json
  // API response:
  { "name": "The Cycle of Thorns", "month_names": null, "season_names": null, "year_count": null }

  // MongoDB document:
  { "name": "The Cycle of Thorns",
    "months": [
      { "name": "Bloomrise", "days": 30 }, ...
    ],
    "weekdays": ["Sun's Day", ...]
  }
  ```
  The DB stores the correct data, but the API serializes the wrong field name (`month_names` instead of `months[].name`). The Almanac tab will render empty month names even though the data exists.
- **Known gap?** Cluster F2 (modern calendar Gregorian population). Here it's a **fantasy** calendar with the same null serialization bug — the data is in `months[]` but the API looks for `month_names`. This is likely a generic serializer mismatch.

### [SEV: MED] Cluster G — Recap returns null `recap_text`, `current_place`, `when`
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** `GET /chronicle/recap/:iid` after 30+ turns
- **Expected vs got:** Expected: a non-null recap_text string + current_place + when. Got: `recap_text: null`, `current_place: null`, `when: null`. Only `open_threads` array is populated.
- **Evidence:**
  ```json
  { "recap_text": null, "current_place": null, "when": null, "open_threads": [ ... 5 threads ... ] }
  ```
  The Recap tab (Lore Tome landing) would render empty for a 30-turn world. The `open_threads` survive, but the prose spine and place/time are missing.
- **Known gap?** Cluster G — reported in the merged findings. Still live and reproducing.

### [SEV: MED] Cluster D — Location duplicates: "the keep" vs "Thornhaven" (article variant)
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** After 10+ turns with movement, `GET /chronicle/locations/:iid`
- **Expected vs got:** Expected: the keep's battlements and the great hall are under one Thornhaven node. Got: `places` list contains both "the keep" and "the battlements of Thornhaven" as separate root-level nodes (no `parent_id` set), plus "the war room", "the great hall", "the chapel" — all with `place_kind: null`, suggesting the Location Graph P1 spine (`parent_id`/`world_root_id`) is not being populated for new locations.
- **Evidence:**
  ```json
  { "name": "the keep", "kind": null, "parent": null }
  { "name": "the battlements of Thornhaven", "kind": null, "parent": null }
  { "name": "the war room", "kind": null, "parent": null }
  ```
  All locations are flat; no nesting. The `place_kind` field is never set, and `parent_id` is never populated. The atlas tree (P2) will render as a flat list, not a nested tree.
- **Known gap?** Cluster D — location fragmentation. The `vague-label guard` and `fuzzy resolution` work, but the **cartographer** (`placeLocation`) is not assigning `parent_id` or `place_kind` to new locations in this instance. This is a deeper failure than article variants — the spine is not being built.

### [SEV: MED] Cluster D — Deeper sub-room not minted (keep's rooms are root-level)
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** Travel from "the war room" → "the great hall" → "the chapel" → "the stables". All are inside the keep.
- **Expected vs got:** Expected: rooms are nested under the keep building. Got: every room is a root-level node with `parent: null`.
- **Evidence:** `GET /chronicle/locations/:iid` returned all places as flat siblings with no parent.
- **Known gap?** Cluster D4 — deeper sub-room under-minting. The P1 cartographer is failing to establish the `parent_id` spine.

### [SEV: MED] Cluster F — `/continue season` lands wrong month name in prose
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** `/continue season` after month 1 → calendar correctly advances to month 4 (day 2). Check the generated narrative.
- **Expected vs got:** Expected: prose should reference the themed month name ("Thornfall" for month 4). Got: prose said "The bruised twilight of the Wildwood bleeds into the gold of **autumn**, then fades further into the skeletal, frost-bitten silence of **winter**." The narrator used generic Gregorian-season words, not the themed calendar.
- **Evidence:** Narrative text from seq 31: "...gold of autumn...frost-bitten silence of winter..." The calendar says month 4 = "Thornfall", but the LLM never saw the month names.
- **Known gap?** Cluster F3 — narrator invents month names not in the calendar. Here the calendar has valid names in DB, but the API returns `null` to the prompt builder, so the LLM has no month names to ground on and falls back to generic seasons.

### [SEV: MED] Presence — absent NPCs flicker in and out on recall questions
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** Ask "What is my sister's name?" (Mira is absent). Check `present_characters` on the next turn.
- **Expected vs got:** Expected: Mira should NOT appear in `present_characters` because she is absent. Got: Mira never appeared in `present_characters` in this run, but the "Mira" card was minted in the Bonds ledger. The main issue is that the **absent character was carded** (see Cluster B above), not a false-presence flicker.
- **Evidence:** Not a direct presence flicker, but the absent-relative carding is a related semantic corruption.
- **Known gap?** Cluster C — presence conflates memory-recall with physical co-location. Not directly observed in this run, but the absent-relative carding is a correlated failure.

### [SEV: MED] Rewind audit: `removed_counts` are null in HTTP response
- **World/instance:** "Thornhaven" iid=`6a2bd57cafc85d8941c370ea`
- **Repro:** `POST /chronicle/rewind/:iid {"sequence":30}`
- **Expected vs got:** Expected: `removed_events`, `removed_memories`, `removed_entities` should show counts. Got: all three fields are `null` in the JSON response.
- **Evidence:**
  ```json
  { "success": true, "removed_events": null, "removed_memories": null, "removed_entities": null }
  ```
  The rewind actually succeeded (events dropped from 43 to 28), but the response doesn't report the counts. This is a UX/api bug, not data corruption.
- **Known gap?** NEW — minor API response bug.

---

## Still-Open Focus (§4.5) — Evidence Summary

| Open Item | Status | Evidence |
|---|---|---|
| **A — side-chat leak to Threads/Recap** | **CONFIRMED (new surface)** | Secret "fae ink" atom with `origin:side_chat` surfaces in `GET /chronicle/memories?q=fae%20ink`. Threads/Recap clean in this run, but Echoes search is a leak surface. |
| **B1 — sentient/character identity attribution** | N/A (GM lane) | GM lane hit the B2 variant: protagonist epithet "Sentinel" carded as NPC. |
| **C — presence conflates recall with co-location** | Not directly observed | No false-present flicker in this run, but absent-relative carding is a related semantic corruption. |
| **D — location duplicates + sub-rooms not minted** | **CONFIRMED** | All locations are flat root-level nodes; `parent_id` and `place_kind` are null. The cartographer is not building the spine. |
| **E — memory version-linking on correction** | **PASS** | Mara→Mira: old atoms superseded, `updates_memory_ids` linked. Verified live. |
| **F — themed-calendar grounding + /continue season month** | **CONFIRMED** | `month_names` null in API; prose used "autumn/winter" instead of "Thornfall". |
| **I — event-edit unchanged guard** | **FAIL (regression)** | Unchanged ai_response returns 200 + recuration instead of 400. |

---

## New Bugs Found

1. **Echoes search leaks side-chat atoms** (Cluster A, new surface) — `GET /chronicle/memories?q=` does not filter `origin:side_chat` before returning results. The atom is correctly tagged, but the search endpoint doesn't exclude it.
2. **GM-world protagonist epithet carded** (Cluster B variant) — "Sentinel" / "the traveler" / "the rider" epithets used by the narrator are minted as separate NPC cards, fragmenting the protagonist identity. This is the GM-world sibling of the sentient-world B2 bug.
3. **Calendar API serialization mismatch** (Cluster F2, generic) — DB stores `months[]`, API returns `month_names: null`. The Almanac and prompt builder both receive null, so the LLM has no themed months to ground on.
4. **Cartographer not assigning `parent_id`/`place_kind`** (Cluster D variant) — New locations in a fresh instance are minted with `parent_id: null` and `place_kind: null`, so the nested atlas (P2) renders flat. The P1 spine is not being populated.
5. **Rewind response hides counts** (NEW, low) — `removed_events/memories/entities` are null in the HTTP response despite the rewind actually working.

---

## §5 Audit Results

| Audit | Result |
|---|---|
| `audit:location` | ✅ ALL INVARIANTS HELD |
| `audit:movement` | ✅ ALL GREEN (45/45) |
| `audit:location-resolution` | ✅ ALL INVARIANTS HELD |
| `audit:memory-links` | ✅ ALL GREEN (9/9) |
| `continuity-audit` (live instance) | ✅ 8/8 ok |

---

## §5 Chronicle Endpoints Hit

| Endpoint | Result | Notes |
|---|---|---|
| `GET /chronicle/recap/:iid` | Hit | `recap_text` null, `open_threads` 5 items |
| `GET /chronicle/events/:iid` | Hit | 43 events before rewind, 28 after |
| `GET /chronicle/memories/:iid?q=` | Hit | 4 results for "Mira"; 2 for "fae ink" (1 side-chat leak) |
| `GET /chronicle/calendar/:iid` | Hit | `month_names: null`, `story_calendar.month: 4` |
| `GET /chronicle/threads/:iid` | Hit | `threads: null` (no unresolved main threads) |
| `GET /chronicle/relationships/:iid` | Hit | 9 characters, including "Sentinel", "Mara", "Mira", "Stranger" + "the stranger" |
| `GET /chronicle/relationships/:iid/:charId/memories` | Hit | 2 memories for Stranger |
| `GET /chronicle/locations/:iid` | Hit | 7+ places, all flat, `kind: null`, `parent: null` |
| `GET /chronicle/side-chats/:iid` | Hit | 1 thread (Stranger, 2 turns) |
| `PUT /chronicle/event/:id` | Hit | Wrong field 400 ✅; unchanged 200 ❌; changed 200 ✅ |
| `POST /chronicle/rewind/:iid` | Hit | Success true, counts null, events 43→28 |

---

## Session / Dev-State Mutations

- `redis-cli del rl:template_create:6a210ba38e6db660dc8ef6a3` — **NOT needed** (quota showed 84 remaining).
- `redis-cli del session:6a2bd57cafc85d8941c370ea` — executed after rewind.
- No other dev-state mutations.
- World and instance **persisted** (not deleted).

---

## Summary

- **Template:** `6a2bd56dafc85d8941c370d3` ("Thornhaven")
- **Instance:** `6a2bd57cafc85d8941c370ea`
- **Regression checks:** 2/3 PASS (wrong field 400 ✅, unchanged 400 ❌, changed 200 ✅)
- **Key corruption-class bugs:**
  1. **Identity boundary collapse** (HIGH): Protagonist epithet "Sentinel" carded as NPC; absent sister "Mara" / "Mira" carded as two separate NPCs.
  2. **Duplicate character cards** (HIGH): "Stranger" vs "the stranger" — same character, two cards.
  3. **Side-chat leak in Echoes search** (HIGH): `GET /chronicle/memories?q=fae%20ink` surfaces the private side-chat atom.
  4. **Calendar API null** (MED): `month_names` null → LLM uses generic seasons instead of themed months.
  5. **Location spine not built** (MED): All places flat, `parent_id`/`place_kind` null.
  6. **Recap null** (MED): `recap_text`, `current_place`, `when` all null after 30+ turns.
  7. **Rewind counts hidden** (LOW): Response returns null for removed counts.
