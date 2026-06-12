# Playtest Findings — 2026-06-12c — Agent 2

**QA Agent:** #2  
**Lane:** GM world, high fantasy — "Thornhaven"  
**Template ID:** `6a2bd56dafc85d8941c370d3`  
**New instance ID:** `6a2bea82626fb837070f2b69` (created this run)  
**Existing instance ID:** `6a2bd57cafc85d8941c370ea` (continuity-audit only)  
**World kept:** YES (per lifecycle policy)

---

## Regression Checks (fix-batch verify)

| Check | Result | RAW proof |
|---|---|---|
| **Event-edit wrong field → HTTP 400** | **PASS** | `PUT /chronicle/event/6a2beb927c5c71f2912fef3d {"narrative":"x"}` → `HTTP 400` `{"error":"No editable event fields provided. Use ai_response and/or player_input."}` |
| **Event-edit unchanged ai_response → HTTP 400** | **PASS** | Copied exact `data.ai_response` from seq 18 event `6a2beb927c5c71f2912fef3d` (707 chars) via `GET /chronicle/events/:iid`, PUT back unchanged → `HTTP 400` `{"error":"Event edit did not change ai_response or player_input."}` |
| **Event-edit empty string → HTTP 400** | **PASS** | `PUT {"ai_response":""}` → `HTTP 400` `{"error":"ai_response cannot be empty."}` |
| **Event-edit changed ai_response → HTTP 200** | **PASS** | Changed text → `HTTP 200` `{"success":true,"recuration_queued":true,...}` |
| **Themed-calendar `month_names` serialized** | **PASS** | `GET /chronicle/calendar/6a2bea82626fb837070f2b69` → `"month_names":["Frostbloom","Sunveil","Harvestshade","Glimmergale","Moonshadow","Faeweave","Thornfire","Everbloom"]` (was `null` in 2026-06-12b run) |
| **Recap `recap_text` non-null** | **PASS** | Before edit regression: `GET /chronicle/recap/:iid` → `"recap_text": "*The wind howls across the ramparts, whipping a stray lock of hair across your face..."`, `"current_place":"the battlements"`, `"when":"a season"`, `"open_threads_count":5` |
| **Location duplicate-node dedup** | **FAIL** | API + Mongo both show duplicate location entities (see Cluster D below) |

**Score: 6/7 PASS.** Unchanged-edit guard, empty-string guard, calendar serializer, and recap generation all landed. Location dedup did not.

---

## Corruption-Class Bugs (P0 / P1)

### [SEV: HIGH] Cluster D — Location duplicate nodes persist (dedup fix did not land)
- **World/instance:** GM world "Thornhaven" iid=`6a2bea82626fb837070f2b69`
- **Repro:** Play 18 main turns with movement (battlements → war room → great hall → chapel → Wildwood → Thornhaven stables → battlements). Then `GET /chronicle/locations/:iid` and Mongo `entities` query.
- **Expected vs got:** Expected: one node per canonical place (Wildwood, Thornhaven, the battlements, the war room). Got: duplicate nodes for each.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/locations/:iid — name frequency:
  [{"name":"Wildwood","count":3},{"name":"Thornhaven","count":2},
   {"name":"the battlements","count":2},{"name":"the war room","count":2}]

  // Mongo entities (instance 6a2bea82626fb837070f2b69), duplicates by canonical_name:
  [{"name":"wildwood","count":3},{"name":"thornhaven","count":2},
   {"name":"the battlements","count":2},{"name":"the war room","count":2}]
  ```
  Some duplicates have `parent_id` set (partial spine), but canonical dedup still mints parallel nodes — e.g. two "the battlements" entities, one with `parent_id:null`, one with `parent_id:"6a2beb5c7c5c71f2912feefb"`.
- **Verified how:** `GET /chronicle/locations/:iid` + Mongo `db.entities.find({instance_id})`
- **Known gap?** Cluster D — location fragmentation / duplicate-node dedup (merged findings §3, fix queue #4)

### [SEV: HIGH] Cluster A / N6 — Echoes search still lacks side-chat privacy gate (sanity check)
- **World/instance:** GM world "Thornhaven" iid=`6a2bea82626fb837070f2b69`
- **Repro:** `/side 6a2beb297c5c71f2912feec2 "I know something about the raven you don't. The scroll was written in fae ink."` → `GET /chronicle/memories/:iid?q=fae%20ink`
- **Expected vs got:** Endpoint should filter `origin:'side_chat'` atoms from main search (real leak proof requires sentient world where protagonist ∉ `known_by`). In this GM sanity check: atom is correctly tagged but **still returned by search**.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/memories/6a2bea82626fb837070f2b69?q=fae%20ink
  {"side_chat_leaked":true,"side_chat_count":1,"items":[{
    "origin":"side_chat",
    "text":"The player revealed to Stranger that the scroll was written in fae ink, suggesting a deeper connection to the fae realm..."
  }]}
  ```
- **Verified how:** `GET /chronicle/memories/:iid?q=fae%20ink`
- **Known gap?** Cluster A / N6 — Echoes search bypasses privacy gate. **Still open.** Note: in GM worlds, Threads/Recap surfacing for the protagonist is by-design (playbook §7); this finding is about the **search endpoint lacking the gate**, not a GM-world leak claim.

### [SEV: MED] Cluster B2b — Absent relative "Mira" minted as Bonds card
- **World/instance:** iid=`6a2bea82626fb837070f2b69`
- **Repro:** Mention absent sister Mira in main chat; `GET /chronicle/relationships/:iid`
- **Expected vs got:** Expected: no codex card for a never-present relative. Got: **Mira** card in Bonds ledger.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/relationships/6a2bea82626fb837070f2b69
  {"characters":[
    {"id":"6a2beb297c5c71f2912feec2","name":"Stranger","role":"unknown"},
    {"id":"6a2bead17c5c71f2912fee43","name":"Mira","role":"unknown"},
    {"id":"6a2bea967c5c71f2912fedde","name":"Raven","role":"messenger"}
  ]}
  ```
  Mongo also has entity `{canonical_name:"Mira"}` with no `present_characters` co-location.
- **Verified how:** `GET /chronicle/relationships/:iid` + Mongo entities
- **Known gap?** Cluster B2b — player-card guard covers player persona, not absent relatives (merged findings §1)

### [SEV: MED] Cluster F3 — `/continue season` prose uses generic seasons despite populated calendar
- **World/instance:** iid=`6a2bea82626fb837070f2b69`
- **Repro:** `/continue season` after 16 turns; calendar API now returns themed `month_names`.
- **Expected vs got:** Expected: prose references themed month names (e.g. "Thornfire", "Frostbloom"). Got: generic Gregorian-season language.
- **Evidence (RAW):** Seq 17 narrative (from agent-chat log):
  > *"Autumn bleeds into a harsh, skeletal winter... The first thaw of spring arrives not with warmth..."*
  Calendar at seq 17: `story_calendar.month: 4`, `month_names` populated in API.
- **Verified how:** agent-chat frame dump seq 17 + `GET /chronicle/calendar/:iid`
- **Known gap?** Cluster F3 — narrator calendar grounding. Serializer fix landed; prompt builder may still not inject month names into generation context.

---

## Still-Open Focus (§4.5) — Evidence Summary

| Open Item | Status | Evidence |
|---|---|---|
| **A — Echoes search side-chat gate** | **STILL OPEN (sanity confirmed)** | `origin:"side_chat"` atom returned by `GET /chronicle/memories?q=fae%20ink`. Real leak proof needs sentient world. |
| **B2b — absent relative carding** | **CONFIRMED** | Mira card minted; never on-scene. |
| **D — location duplicate nodes** | **CONFIRMED (dedup fix FAIL)** | Wildwood×3, Thornhaven×2, battlements×2, war room×2 in API + Mongo. |
| **F — calendar month_names** | **CLOSED (this run)** | API returns 8 themed month names. |
| **G — recap null fields** | **CLOSED (this run)** | `recap_text`, `current_place`, `when` all populated after 18 turns. |
| **I-b — unchanged edit guard** | **CLOSED (this run)** | Exact-text PUT → 400. |

---

## Improvements vs 2026-06-12b (same lane)

| Item | 2026-06-12b | 2026-06-12c |
|---|---|---|
| Unchanged edit → 400 | FAIL | **PASS** |
| Empty edit → 400 | FAIL (#6) | **PASS** |
| `month_names` in API | `null` | **populated array** |
| `recap_text` | `null` | **non-null prose** |
| "Stranger" vs "the stranger" duplicate cards | 2 cards | **1 card** (Stranger only) |
| Location `parent_id` spine | all null | **partial** (chapel, great hall, quarters have parent; duplicates still minted) |
| Echoes side-chat in search | leaked | **still leaks** (gate not fixed) |

---

## §5 Audit Results

| Audit | Result |
|---|---|
| `audit:location` | ✅ ALL INVARIANTS HELD |
| `audit:movement` | ✅ ALL GREEN (45/45) |
| `audit:location-resolution` | ✅ ALL INVARIANTS HELD |
| `audit:memory-links` | ✅ ALL GREEN (9/9) |
| `continuity-audit` (existing `6a2bd57cafc85d8941c370ea`) | ✅ 8/8 ok, `healthy:true` |
| `continuity-audit` (new `6a2bea82626fb837070f2b69`) | ✅ 8/8 ok, `healthy:true` |

**Existing instance continuity-audit (RAW):**
```json
{"instanceId":"6a2bd57cafc85d8941c370ea","healthy":true,"summary":{"ok":8,"warn":0,"fail":0},
 "checks":[
   {"name":"event_sequence_integrity","status":"ok","detail":"31 events, sequences contiguous."},
   {"name":"memory_entity_refs","status":"ok","detail":"23 active memories; all entity refs resolve."},
   {"name":"location_cursor","status":"ok","detail":"Current place: the Wildwood."}
 ]}
```

---

## §5 Chronicle Endpoints Hit

| Endpoint | Result | Notes |
|---|---|---|
| `GET /chronicle/recap/:iid` | Hit | `recap_text` ✅, `current_place` ✅, `when` ✅, 5 open_threads |
| `GET /chronicle/events/:iid` | Hit | 18 main events (+ 1 side-chat seq 14) |
| `GET /chronicle/memories/:iid?q=fae%20ink` | Hit | 4 results; 1 `origin:side_chat` (gate missing) |
| `GET /chronicle/memories/:iid?q=Mira` | Hit | 2 active atoms; no Mara supersession chain (correction atom only) |
| `GET /chronicle/calendar/:iid` | Hit | `month_names` populated ✅; `season_names:[]`, `year_count:null` |
| `GET /chronicle/threads/:iid` | Hit | 9 open threads; side-chat fae-ink fact appears (by-design in GM world) |
| `GET /chronicle/relationships/:iid` | Hit | 3 characters: Stranger, Mira, Raven |
| `GET /chronicle/locations/:iid` | Hit | 15 place entries with duplicates |
| `GET /chronicle/side-chats/:iid` | Hit | 1 thread (Stranger, 1 turn) |
| `PUT /chronicle/event/:id` | Hit | wrong 400 ✅; unchanged 400 ✅; empty 400 ✅; changed 200 ✅ |
| `GET /admin/instances/6a2bd57cafc85d8941c370ea/continuity-audit` | Hit | 8/8 ok |

---

## Session / Dev-State Mutations

- **Created instance:** `6a2bea82626fb837070f2b69` (from template `6a2bd56dafc85d8941c370d3`)
- **Played:** 18 main turns + 1 side-chat turn via `agent-chat.ts`
- **Edited event seq 19** during regression testing (changed then re-tested unchanged on seq 18)
- `redis-cli del session:...` — **NOT needed** (no rewind performed)
- No template_create quota reset; no rate-limit edits
- World and instance **persisted** (not deleted)

---

## Summary

- **Template:** `6a2bd56dafc85d8941c370d3` ("Thornhaven")
- **New instance:** `6a2bea82626fb837070f2b69`
- **Regression checks:** **6/7 PASS** — unchanged-edit 400 ✅, empty-edit 400 ✅, wrong-field 400 ✅, month_names ✅, recap_text ✅, location dedup ❌
- **Top findings:**
  1. **Location duplicate nodes still minted** (HIGH) — Wildwood×3, Thornhaven×2, etc.; dedup fix did not land.
  2. **Echoes search side-chat gate still open** (HIGH, sanity) — `origin:side_chat` returned by search; real leak needs sentient world.
  3. **Absent relative Mira carded** (MED) — B2b still open.
  4. **Calendar serializer fixed** but `/continue season` prose still uses generic autumn/winter (MED).
  5. **Fixes confirmed:** unchanged-edit guard, recap generation, month_names API serialization, character variant dedup (single Stranger card).
