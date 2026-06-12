# Playtest Findings — 2026-06-12d — Agent 2

**QA Agent:** #2  
**Lane:** GM fantasy — "Thornhaven"  
**Template ID:** `6a2bd56dafc85d8941c370d3`  
**New instance ID:** `6a2bfb0830b7d5f1412cbf09` (created this run)  
**World kept:** YES (per lifecycle policy)  
**Play log:** `/tmp/agent2_2026-06-12d.log` (14 main turns + `/continue day` + `/continue season`, seq 2–16)

---

## Fix Batch 3 — Live Verification

| Check | Result | RAW proof |
|---|---|---|
| **B2a — player self-intro does NOT mint codex card** | **PASS** | Turn 1 `"I am Seraphine, disgraced knight..."` (seq 2): agent-chat log shows **no** `[codex]` line. After full run, Mongo `db.characters.find({instance_id})` → exactly 3 cards: `Seraphine` (`is_protagonist:true`, template opening), `the groom`, `the raven`. No second Seraphine / no `"The Player"` card. `GET /chronicle/relationships/6a2bfb0830b7d5f1412cbf09` → only groom + raven (protagonist correctly excluded from Bonds NPC list). |
| **B2b — absent relative Mira NOT carded** | **PASS** | Mentioned Mira in seq 4 + seq 14. `GET /chronicle/relationships/6a2bfb0830b7d5f1412cbf09` → `{"characters":[{"name":"the groom",...},{"name":"the raven",...}]}` — **no Mira**. Mongo: entity `{_id:"6a2bfb405f863f8b449c0714","type":"character"}` exists for memory linking, but `db.characters` has **no** Mira doc (`MIRA CHARACTER CARD: null`). Memories still captured: `"Seraphine's sister Mira taught her to read fae script..."`. |
| **Cluster D — location dedup (article/parent variants)** | **PARTIAL / FAIL** | Improved vs run c (Wildwood ×3 → ×2) but duplicates persist. See finding below. |
| **R4 — `/continue season` prose uses themed months** | **FAIL** | Calendar populated; prose still generic Gregorian seasons. See finding below. |

**Score: 2 PASS, 1 PARTIAL/FAIL, 1 FAIL** on fix-batch-3 focus items.

---

## Regression Checks (prior batches — spot-check)

| Check | Result | RAW proof |
|---|---|---|
| **Event-edit wrong field → HTTP 400** | **PASS** | `PUT /chronicle/event/6a2bfb1e5f863f8b449c06db {"narrative":"wrong key"}` → `HTTP 400` `{"error":"No editable event fields provided. Use ai_response and/or player_input."}` |
| **Event-edit unchanged ai_response → HTTP 400** | **PASS** | Copied exact `data.ai_response` (1497 chars) from seq 16 event `6a2bfcd75f863f8b449c086b` via `GET /chronicle/events/:iid`, PUT back unchanged → `HTTP 400` `{"error":"Event edit did not change ai_response or player_input."}` |
| **Event-edit empty string → HTTP 400** | **PASS** | `PUT {"ai_response":""}` → `HTTP 400` `{"error":"ai_response cannot be empty."}` |
| **Themed-calendar `month_names` serialized** | **PASS** | `GET /chronicle/calendar/6a2bfb0830b7d5f1412cbf09` → `"month_names":["Frostbloom","Thawing","Suncrest","Glimmerfall","Harvestmoon","Wyrmrest","Shadowveil","Frostfire"]` |
| **Recap fields populated** | **PASS** | `GET /chronicle/recap/:iid` → `"recap_text": "*The bruised shadows of autumn deepen..."`, `"current_place":"battlements"`, `"when":"a season"` |
| **Continuity audit** | **PASS** | `GET /admin/instances/6a2bfb0830b7d5f1412cbf09/continuity-audit` → `"healthy":true`, 8/8 ok, `"single_protagonist":"Exactly one protagonist card (Seraphine)."` |

---

## Corruption-Class / P1 Bugs

### [SEV: HIGH] Cluster D — Location duplicate nodes persist (partial improvement)
- **World/instance:** GM fantasy "Thornhaven" iid=`6a2bfb0830b7d5f1412cbf09`
- **Repro:** 14-turn movement loop (battlements → war room → great hall → chapel/courtyard → borderlands → Wildwood ×2 visits → Thornhaven stables → battlements). Log: `/tmp/agent2_2026-06-12d.log`
- **Expected vs got:** One entity per canonical place. Got parallel nodes for battlements, Thornhaven, Wildwood, borderlands.
- **Evidence (RAW):**
  ```json
  // Mongo db.entities (type location), canonical_name counts:
  [{"name":"battlements","count":2},{"name":"borderlands","count":2},
   {"name":"thornhaven","count":2},{"name":"wildwood","count":2}]

  // GET /chronicle/locations/6a2bfb0830b7d5f1412cbf09 — duplicate highlights:
  {"name":"battlements","entities":[
    {"entity_id":"6a2bfc6b5f863f8b449c0826","event_count":4},
    {"entity_id":"6a2bfb315f863f8b449c06ff","event_count":1}
  ]}
  {"name":"Wildwood","entities":[
    {"entity_id":"6a2bfc1c5f863f8b449c07d6","event_count":2},
    {"entity_id":"6a2bfb915f863f8b449c0760","event_count":0}
  ]}
  {"name":"Thornhaven","entities":[
    {"entity_id":"6a2bfb495f863f8b449c0721","event_count":0},
    {"entity_id":"6a2bfc555f863f8b449c080f","event_count":1}
  ]}
  ```
  **Improvement vs run c:** Wildwood dropped from ×3 to ×2; war room / great hall now deduped to ×1 (were ×2 in run c). Dedup fix partially landed; not closed.
- **Verified how:** Mongo `db.entities` + `GET /chronicle/locations/:iid`
- **Known gap?** Cluster D — location fragmentation (matrix row D, still open across a→c→d)

### [SEV: MED] Cluster R4 — `/continue season` prose ignores themed month names
- **World/instance:** iid=`6a2bfb0830b7d5f1412cbf09`
- **Repro:** `/continue season` at seq 16 after 14 main turns + `/continue day`
- **Expected vs got:** Expected prose referencing themed months (e.g. "Glimmerfall", "Frostbloom"). Got generic Gregorian-season language despite populated calendar.
- **Evidence (RAW):** Seq 16 narrative (agent-chat log):
  > *"The bruised shadows of **autumn** deepen into the oppressive, white silence of **winter**... As the first thaw of **spring** begins to drip from the eaves..."*
  Calendar at seq 16:
  ```json
  {"current":{"year":1,"month":4,"day":2,"era":"Era of Pacts","label":"a season"},
   "month_names":["Frostbloom","Thawing","Suncrest","Glimmerfall","Harvestmoon","Wyrmrest","Shadowveil","Frostfire"]}
  ```
  Month 4 = **Glimmerfall** — never named in prose.
- **Verified how:** agent-chat seq 16 dump + `GET /chronicle/calendar/:iid`
- **Known gap?** Cluster R4 — calendar prose grounding (matrix row R4)

### [SEV: LOW] Cluster D — Vague opening location "the cold stone" minted
- **World/instance:** iid=`6a2bfb0830b7d5f1412cbf09`
- **Repro:** Opening turn prose mentions "cold stone" parapet; seq 2–3 `location_anchor.name` = `"the cold stone"` before cursor moves to `"battlements"` at seq 4.
- **Expected vs got:** Expected cursor on battlements/Thornhaven from opening. Got a vague-label entity persisted with 2 events.
- **Evidence (RAW):** Mongo: `{"canonical_name":"the cold stone","count":1}` with `event_count:2` in Places API. Seq 2–3 location lines in log: `location: the cold stone`.
- **Verified how:** agent-chat log + `GET /chronicle/locations/:iid`
- **Known gap?** Location vague-label guard (audit:location-resolution covers synthetic cases; live opening still mints)

---

## Improvements vs 2026-06-12c (same lane)

| Item | 2026-06-12c | 2026-06-12d |
|---|---|---|
| **B2b Mira Bonds card** | FAIL (Mira in relationships) | **PASS** (no Mira card) |
| **B2a player self-intro card** | not explicitly tested | **PASS** (no duplicate Seraphine card) |
| Wildwood duplicate count | ×3 | **×2** (partial) |
| battlements duplicate count | ×2 | **×2** (unchanged) |
| war room duplicate count | ×2 | **×1** (fixed) |
| `/continue season` themed months | FAIL (generic seasons) | **FAIL** (still generic) |
| Side-chat exercised | yes (Stranger) | **no** (not in this run) |

---

## §5 Audit Results

| Audit | Result |
|---|---|
| `bun run audit:location` | ✅ ALL INVARIANTS HELD (synthetic) |
| `bun run audit:location-resolution` | ✅ ALL INVARIANTS HELD |
| `bun run audit:codex-dedup` | ✅ ALL INVARIANTS HELD (synthetic) |
| `bun run audit:memory-links` | ✅ ALL GREEN — passed 9, failed 0 |
| `GET /admin/instances/6a2bfb0830b7d5f1412cbf09/continuity-audit` | ✅ 8/8 ok, `healthy:true` |

Live instance still shows location duplicate nodes despite green structural suite (same seam as runs a→c).

**Continuity audit (RAW):**
```json
{"instanceId":"6a2bfb0830b7d5f1412cbf09","healthy":true,"maxSequence":16,
 "summary":{"ok":8,"warn":0,"fail":0},
 "checks":[
   {"name":"single_protagonist","status":"ok","detail":"Exactly one protagonist card (Seraphine)."},
   {"name":"location_cursor","status":"ok","detail":"Current place: battlements."},
   {"name":"time_cursor","status":"ok","detail":"Cursor at seq 16 (≤ 16)."}
 ]}
```

---

## §5 Chronicle Endpoints Hit

| Endpoint | Result | Notes |
|---|---|---|
| `GET /chronicle/recap/:iid` | Hit | `recap_text` ✅, `current_place: battlements`, `when: a season` |
| `GET /chronicle/events/:iid` | Hit | 16 main events, contiguous seq 1–16 |
| `GET /chronicle/memories/:iid?q=Mira` | Hit | 2+ active atoms about Mira; correctly attributed to Seraphine |
| `GET /chronicle/calendar/:iid` | Hit | `month_names` populated ✅; cursor at month 4 (Glimmerfall) after season skip |
| `GET /chronicle/threads/:iid` | Hit | threads returned |
| `GET /chronicle/relationships/:iid` | Hit | 2 NPC cards (groom, raven); **no Mira** ✅ |
| `GET /chronicle/locations/:iid` | Hit | duplicate nodes — dedup partial fail |
| `GET /chronicle/side-chats/:iid` | ⬛ | Not exercised this run |
| `PUT /chronicle/event/:id` | Hit | wrong 400 ✅; unchanged 400 ✅; empty 400 ✅ |
| `GET /admin/instances/:iid/continuity-audit` | Hit | 8/8 ok |

---

## Session / Dev-State Mutations

- **Created instance:** `6a2bfb0830b7d5f1412cbf09` (from template `6a2bd56dafc85d8941c370d3`)
- **Played:** 14 main turns + 2 continue turns via `agent-chat.ts` (~7.5 min)
- **Edited event seq 2** during early regression probe (accidental; seq 16 unchanged test used for authoritative I-b proof)
- `redis-cli del session:...` — **NOT needed** (no rewind performed)
- No template_create quota reset; no rate-limit edits
- World and instance **persisted** (not deleted)

---

## Summary

- **Template:** `6a2bd56dafc85d8941c370d3` ("Thornhaven")
- **New instance:** `6a2bfb0830b7d5f1412cbf09`
- **Fix batch 3:** B2a ✅ PASS, B2b ✅ PASS (Mira no longer carded — major win vs run c), location dedup 🟡 partial (Wildwood ×3→×2, war room fixed, battlements/Thornhaven/borderlands still dup), `/continue season` themed months ❌ FAIL
- **Prior batch regressions:** I-b edit guards ✅, calendar serializer ✅, recap ✅ — all holding
- **Top open items this lane:** Cluster D location dedup (chronic), Cluster R4 calendar prose grounding
