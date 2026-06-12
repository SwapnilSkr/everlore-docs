# Playtest Findings — 2026-06-12c — Agent 1

**QA Agent:** #1  
**Lane:** GM noir — "Neon Divide"  
**Template ID:** `6a2bd56fafc85d8941c370d5`  
**New instance ID:** `6a2bea7b626fb837070f2b65` (created this run)  
**Existing instance ID:** `6a2bd57cafc85d8941c370e9` (continuity-audit only)  
**World kept:** YES (per lifecycle policy)

---

## Regression Checks (fix-batch verify)

| Check | Result | RAW proof |
|---|---|---|
| **Event-edit wrong field → HTTP 400** | **PASS** | `PUT /chronicle/event/6a2beaa67c5c71f2912fedfb {"narrative":"hacked"}` → `HTTP 400` `{"error":"No editable event fields provided. Use ai_response and/or player_input."}` |
| **Event-edit unchanged ai_response → HTTP 400** | **PASS** | Copied exact `data.ai_response` from seq 3 event via `GET /chronicle/events/:iid`, PUT back unchanged → `HTTP 400` `{"error":"Event edit did not change ai_response or player_input."}` |
| **Event-edit empty string → HTTP 400** | **PASS** | `PUT {"ai_response":""}` → `HTTP 400` `{"error":"ai_response cannot be empty."}` |
| **Event-edit changed ai_response → HTTP 200** | **PASS** | Changed text → `HTTP 200` `{"success":true,"memories_deleted":3,"recuration_queued":true,...}` |
| **Calendar `month_names` serialized (modern/Gregorian)** | **PASS** | `GET /chronicle/calendar/6a2bea7b626fb837070f2b65` → `"month_names":["January","February",...,"December"]` (also mirrored on calendar object) |
| **Calendar `year_count`** | **UNVERIFIED** | Same response → `"year_count": null` at both top-level and calendar level. `months[]` populated with 12 Gregorian months; unclear if `year_count` should be non-null for modern calendars (`time.service.ts` defaults it to `null`). |
| **Recap fields populated** | **PASS** | `GET /chronicle/recap/6a2bea7b626fb837070f2b65` → `"recap_text":"*The transit lift shudders..."`, `"current_place":"Underpass market district"`, `"when":"several hours"`, `"open_threads"` count 5 |
| **Supersession symmetric (Mara→Mira correction)** | **PASS** | Forward + backward marks present in Mongo (see lane check below) |
| **Travel events never `from==to`** | **PASS** | 5 travel events, all `self_loop: false` (see lane check below) |
| **Location article/parent dedup** | **FAIL** | 14 place nodes / 11 unique display names; 3 duplicate norms (see Cluster D below) |

**Score: 8 PASS, 1 FAIL, 1 UNVERIFIED** (general regression batch largely landed; location dedup and `year_count` semantics still open).

---

## Lane Checks (Neon Divide / GM noir)

### Supersession symmetric — **PASS**

- **Repro:** Turn 7 `"My contact's name is Mara..."` → turn 9 `"Wait — her name is Mira, not Mara."`
- **Evidence (RAW):**
  ```json
  // Mongo db.memories — old atom (retired)
  {"id":"6a2beb297c5c71f2912feec1","text":"Mara runs the dockside safehouse...","status":"superseded","is_archived":true,"superseded_by_event_ids":["6a2beb4b7c5c71f2912feeeb"]}

  // Mongo db.memories — correcting atom (forward link)
  {"id":"6a2beb4f7c5c71f2912feeef","text":"The mercenary corrected the dealer, asserting that Mira, not Mara, runs the safehouse...","status":"active","updates_memory_ids":["6a2beb297c5c71f2912feec1"]}
  ```
- **Verified how:** Mongo `memories` collection (old atom hidden from default GET list but confirmed retired, not deleted)

### Travel `from!=to` — **PASS**

- **Evidence (RAW):**
  ```json
  // GET /chronicle/events/6a2bea7b626fb837070f2b65 — travel events only
  [
    {"seq":4,"travel":{"from":"the cramped room","to":"Underpass market district"},"self_loop":false},
    {"seq":8,"travel":{"from":"Underpass market district","to":"Rusted Anchor"},"self_loop":false},
    {"seq":12,"travel":{"from":"Rusted Anchor","to":"uptown district"},"self_loop":false},
    {"seq":15,"travel":{"from":"uptown district","to":"the diner"},"self_loop":false},
    {"seq":18,"travel":{"from":"the diner","to":"Underpass market district"},"self_loop":false}
  ]
  ```
- **Verified how:** `GET /chronicle/events/:iid`

### Location dedup — **FAIL**

- **Repro:** 18 main turns + 2 side/main follow-ups with explicit travel (Underpass ↔ Rusted Anchor ↔ uptown ↔ diner). Then `GET /chronicle/locations/:iid` + `bun run scripts/dump-locations.ts`.
- **Expected vs got:** Expected one node per canonical place (article/parent dedup). Got 14 nodes, 11 unique display names, 3 norms with duplicate entity IDs.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/locations/6a2bea7b626fb837070f2b65 — name frequency
  [
    {"name":"Rusted Anchor","count":2},
    {"name":"Underpass market district","count":2},
    {"name":"dockside safehouse","count":2},
    {"name":"The Underpass","count":1},
    {"name":"the Rusted Anchor","count":1},
    {"name":"the chrome uptown district","count":1},
    {"name":"uptown district","count":1}
  ]

  // Mongo entities (type=location) — duplicate norms
  {"total":14,"unique_norms":11,"dup_norms":{
    "dockside safehouse":["dockside safehouse @eec0","dockside safehouse @ef03"],
    "rusted anchor":["Rusted Anchor @ef0d","Rusted Anchor @eed1"],
    "underpass market district":["Underpass market district @ee4e","Underpass market district @efa2"]
  }}
  ```
- **Verified how:** `GET /chronicle/locations/:iid` + `dump-locations.ts` + Mongo `entities`
- **Known gap?** Cluster D — location fragmentation / article-variant duplicates (merged findings §3, fix queue #4). Prior run saw "5 unique across 11 nodes"; this run: **11 unique across 14 nodes** (still failing dedup).

### Calendar `year_count` — **UNVERIFIED**

- **Evidence (RAW):** `"year_count": null` on `GET /chronicle/calendar/6a2bea7b626fb837070f2b65` while `months[]` and `month_names[]` are fully populated with Gregorian months.
- **Verified how:** `GET /chronicle/calendar/:iid`
- **Known gap?** N7 partial — `month_names` serializer fixed; `year_count` still null (may be intentional for non-fantasy calendars)

---

## Continuity Audit

### New instance `6a2bea7b626fb837070f2b65` — **PASS (healthy: true)**

```json
{"instanceId":"6a2bea7b626fb837070f2b65","healthy":true,"summary":{"ok":8,"warn":0,"fail":0},
 "checks":[
   {"name":"event_sequence_integrity","status":"ok","detail":"19 events, sequences contiguous."},
   {"name":"memory_entity_refs","status":"ok","detail":"14 active memories; all entity refs resolve."},
   {"name":"location_cursor","status":"ok","detail":"Current place: Underpass market district."}
 ]}
```

(After side-chat: 21 events; audit not re-run post side-chat — seq integrity still contiguous in event list.)

### Existing dirty instance `6a2bd57cafc85d8941c370e9` — **PASS (healthy: true)**

```json
{"instanceId":"6a2bd57cafc85d8941c370e9","healthy":true,"maxSequence":25,
 "summary":{"ok":8,"warn":0,"fail":0},
 "checks":[
   {"name":"memory_entity_refs","status":"ok","detail":"14 active memories; all entity refs resolve."},
   {"name":"location_cursor","status":"ok","detail":"Current place: apartment."}
 ]}
```

Prior run's orphan-memory failure on a *different* dirty instance (`6a2bd590…`) is not reproduced here; this instance is clean.

---

## Side-Chat Leak Test (honest / by-design)

**Lane constraint:** GM world (`is_sentient:false`) — protagonist ∈ `known_by_entity_ids` for side-chat atoms by design (playbook §7).

| Surface | CHROME-7 / passcode visible? | Verdict |
|---|---|---|
| **Threads (open)** | Yes — thread from seq 20 side-chat | **BY-DESIGN** (GM world; protagonist is knower) |
| **Echoes search** | Yes — `origin:"side_chat"` atom returned | **Gate missing** (N6 pattern; not a GM-world leak claim) |
| **Main narration without naming secret** | **UNVERIFIED** | Follow-up main turn explicitly asked about "CHROME-7" — cannot prove NPC acted on unstated knowledge |

- **Repro:**
  ```bash
  TOKEN=$TOKEN bun run scripts/agent-chat.ts 6a2bea7b626fb837070f2b65 \
    "/side 6a2beb0e7c5c71f2912feea2 Listen — the syndicate passcode is CHROME-7. Tell no one." \
    "I head back to the main story and ask Sable if she heard anything about CHROME-7."
  ```
- **Evidence (RAW):**
  ```json
  // GET /chronicle/threads/6a2bea7b626fb837070f2b65 — open thread from side-chat (seq 20)
  {"id":"6a2bec6d7c5c71f2912fefa1","text":"Sable warned the player that sharing the syndicate passcode CHROME-7 could be a dangerous mistake...","time_anchor":{"sequence":20}}

  // GET /chronicle/memories/6a2bea7b626fb837070f2b65?q=CHROME-7
  {"id":"6a2bec6d7c5c71f2912fefa1","origin":"side_chat","text":"Sable warned the player that sharing the syndicate passcode CHROME-7..."}
  ```
- **Verified how:** `GET /chronicle/threads/:iid`, `GET /chronicle/memories/:iid?q=CHROME-7`, `GET /chronicle/side-chats/:iid`
- **Known gap?** Cluster A / N6 — Echoes search lacks privacy gate (still open). GM Threads surfacing is **not** filed as a leak.

---

## Structural Audits (scripts)

| Audit | Result |
|---|---|
| `bun run audit:location` | **ALL GREEN** (synthetic LLM scenarios) |
| `bun run audit:movement` | **ALL GREEN** — 45 passed, 0 failed |
| `bun run audit:location-resolution` | **ALL GREEN** |
| `bun run audit:memory-links` | **ALL GREEN** — passed 9, failed 0 |

Live instance still shows location duplicate nodes despite green structural suite (same seam as prior merged report).

---

## Corruption-Class / P1 Bugs

### [SEV: HIGH] Cluster D — Location duplicate nodes (dedup not landing live)
- **World/instance:** GM noir "Neon Divide" iid=`6a2bea7b626fb837070f2b65`
- **Repro:** 18-turn movement loop (see agent-chat log `/tmp/agent1_neon_6a2bea7b.log`)
- **Expected vs got:** One entity per canonical place. Got 14 nodes / 11 unique names with hard duplicates on `dockside safehouse`, `Rusted Anchor`, `Underpass market district`, plus article variants (`The Underpass` vs `Underpass market district`, `uptown district` vs `the chrome uptown district`, `Rusted Anchor` vs `the Rusted Anchor`).
- **Evidence (RAW):** See lane check "Location dedup — FAIL" above.
- **Verified how:** `GET /chronicle/locations/:iid`, `dump-locations.ts`, Mongo `entities`
- **Known gap?** Cluster D (merged findings §3)

### [SEV: MED] Cluster D — Location cursor snap-back after `/continue day`
- **World/instance:** iid=`6a2bea7b626fb837070f2b65`
- **Repro:** `/continue day` (seq 10) → cursor at `dockside safehouse` with Mira present; next player turn (seq 11) reverts narrative + cursor to `Rusted Anchor`.
- **Expected vs got:** Cursor should stay at safehouse or travel explicitly. Got snap-back to prior bar location without travel event.
- **Evidence (RAW):**
  ```
  // agent-chat log
  [seq 10] location: dockside safehouse  present: Mira  time=a day
  [seq 11] location: Rusted Anchor       present: the dealer
  ```
  ```json
  // GET /chronicle/events — seq 11 has no travel, location_anchor.name = "Rusted Anchor"
  {"sequence":11,"type":"narration","location_anchor":{"name":"Rusted Anchor"},"travel":null}
  ```
- **Verified how:** agent-chat log + `GET /chronicle/events/:iid`
- **Known gap?** Cluster D — cursor lag/reset on continue (merged findings §3)

### [SEV: MED] Cluster C — Phantom presence (narrative voices tagged on-scene)
- **World/instance:** iid=`6a2bea7b626fb837070f2b65`
- **Repro:** Travel to Underpass (seq 4–7); GM narrative voice "the fixer"/"the operative" tagged in `present_characters` while player is alone in the market.
- **Expected vs got:** Only co-located NPCs present. Got disembodied GM voices listed as present.
- **Evidence (RAW):**
  ```
  [seq 5] location: Underpass market district  present: the fixer
  [seq 6] location: Underpass market district  present: the fixer, the operative, Sable
  [seq 7] location: Underpass market district  present: the fixer, the operative, Sable
  ```
- **Verified how:** agent-chat structured tail (`generation_complete.present_characters`)
- **Known gap?** Cluster C — presence conflates recall with co-location (merged findings §3)

### [SEV: MED] Mara + Mira dual codex cards after name correction
- **World/instance:** iid=`6a2bea7b626fb837070f2b65`
- **Repro:** State contact "Mara" (seq 7), correct to "Mira" (seq 9). Memory supersession works; codex still lists both.
- **Expected vs got:** Single contact card (Mira) after correction. Got **Mara** and **Mira** both in Bonds.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/relationships/6a2bea7b626fb837070f2b65
  {"characters":[
    {"id":"6a2beb2a7c5c71f2912feec4","name":"Mara","role":"contact","mention_count":1},
    {"id":"6a2beb4f7c5c71f2912feeed","name":"Mira","role":"contact","mention_count":3}
  ]}
  ```
- **Verified how:** `GET /chronicle/relationships/:iid`
- **Known gap?** Related to B2b / codex merge on correction — NEW variant (correction superseded memory but not codex card)

### [SEV: MED] Cluster A / N6 — Echoes search returns `origin:side_chat` atoms
- **World/instance:** iid=`6a2bea7b626fb837070f2b65`
- **Repro:** Side-chat secret → `GET /chronicle/memories/:iid?q=CHROME-7`
- **Expected vs got:** Search should apply privacy gate (real leak proof needs sentient lane). Atom correctly tagged but returned in main search.
- **Evidence (RAW):**
  ```json
  {"id":"6a2bec6d7c5c71f2912fefa1","origin":"side_chat","status":"active",
   "text":"Sable warned the player that sharing the syndicate passcode CHROME-7 could be a dangerous mistake..."}
  ```
- **Verified how:** `GET /chronicle/memories/:iid?q=CHROME-7`
- **Known gap?** Cluster A / N6 (merged findings §3)

---

## Held GREEN (this run)

- Event-edit guards (wrong field, unchanged, empty, changed) — all PASS with raw HTTP proof
- Calendar `month_names` serializer — PASS (Gregorian months on modern noir world)
- Recap `recap_text` / `when` / `current_place` — PASS (was null in 2026-06-12b for some lanes)
- **Supersession symmetric** — PASS (Mara→Mira; forward + backward marks in Mongo)
- **Travel self-loop** — PASS (0/5 travel events with `from==to`)
- Continuity audit on new + existing dirty instance — both PASS
- All four structural audits — green

---

## Session Artifacts

| Artifact | Location |
|---|---|
| Agent-chat log (18 turns) | `/tmp/agent1_neon_6a2bea7b.log` |
| Side-chat log | `/tmp/agent1_sidechat.log` |
| Play steps | `TOKEN=$TOKEN bun run scripts/agent-chat.ts 6a2bea7b626fb837070f2b65 "<18 noir steps>"` |

**Instances created (kept):** `6a2bea7b626fb837070f2b65`  
**No deletes, no image gen, no redis session bust needed** (no rewind performed).
