# Playtest Findings — 2026-06-12d — Agent 1

**QA Agent:** #1  
**Lane:** GM noir — "Neon Divide"  
**Template ID:** `6a2bd56fafc85d8941c370d5`  
**New instance ID:** `6a2bfafd30b7d5f1412cbeff` (created this run)  
**World kept:** YES (per lifecycle policy)  
**Play log:** `/tmp/agent1-play-6a2bfafd.log` (19 main turns, seq 2–20)

---

## Fix Batch 3 — Live Verification

| Check | Result | RAW proof |
|---|---|---|
| **B2a — player self-intro does NOT mint codex card** | **PASS** | After turn `"I'm Kade, a burned-out fixer..."` (seq 3): `db.characters.find({instance_id})` → no doc with `canonical_name` matching `/kade/i`. `GET /chronicle/relationships/6a2bfafd30b7d5f1412cbeff` lists only NPCs (Bartender, Mara Chen, Courier, Vesper, Stranger, Scarred Man) — no Kade / no player-named card. Protagonist card is pre-existing `Operative` (`is_protagonist:true`, created seq 2 opening, not a self-intro mint). |
| **R2 — codex merge on "X not Y" name correction** | **PASS** | Intro `"Mira Chen"` (seq 9) → correction `"Mara Chen, not Mira"` (seq 10). Mongo: single codex `{"_id":"6a2bfbfd5f863f8b449c07b5","canonical_name":"Mara Chen"}`. No Mira character doc. Old promise memory retired: `{"_id":"6a2bfbbd5f863f8b449c0782","text":"Mira Chen, the player's fixer contact...","status":"superseded","is_archived":true,"superseded_by_event_ids":["6a2bfbfa5f863f8b449c07b0"]}`. Forward link on correction atom: `{"_id":"6a2bfbfe5f863f8b449c07b7","updates_memory_ids":["6a2bfbbd5f863f8b449c0782"]}`. |
| **Cluster D — location dedup (article/parent variants)** | **FAIL** | `GET /chronicle/locations/6a2bfafd30b7d5f1412cbeff` → 13 place nodes for one playthrough. Hard duplicates / orphans (see finding below). |
| **Cluster D-cursor — cursor lag on `/continue` & returns** | **FAIL** | Chat-turn lag at seq 8 (outside prose, cursor stuck). `/continue` turns mostly track narrative (seq 7, 14, 18), but seq 18→19 snap without travel marker (see finding below). |
| **D-loop — travel `from!=to`** | **PASS** | Mongo `events.data.travel` (API omits `travel` field — all null in GET): `[{"seq":5,"from":"a dead-end street","to":"the Rusted Bolt"},{"seq":10,"from":"old-town","to":"the Rusted Bolt"},{"seq":12,"from":"old-town","to":"the Divide"},{"seq":15,"from":"The Zenith District","to":"the Rusted Bolt"}]` — no self-loops. |

**Score: 3 PASS, 2 FAIL, 0 UNVERIFIED** on fix-batch-3 focus items.

---

## Regression Checks (prior batches — spot-check)

| Check | Result | RAW proof |
|---|---|---|
| Calendar `month_names` (Gregorian) | **PASS** | `GET /chronicle/calendar/6a2bfafd30b7d5f1412cbeff` → `"month_names":["January",...,"December"]` |
| Recap fields populated | **PASS** | `GET /chronicle/recap/6a2bfafd30b7d5f1412cbeff` → `"current_place":"the Rusted Bolt"`, `"when":"a day"`, non-empty `recap_text` |
| Supersession symmetric (Mira→Mara) | **PASS** | See R2 row above |
| Continuity audit | **PASS** | `GET /admin/instances/6a2bfafd30b7d5f1412cbeff/continuity-audit` → `"healthy":true`, 8/8 ok |

---

## Structural Audits (scripts)

| Audit | Result |
|---|---|
| `bun run audit:location` | **ALL GREEN** (synthetic LLM scenarios) |
| `bun run audit:movement` | **ALL GREEN** — 45 passed, 0 failed |
| `bun run audit:location-resolution` | **ALL GREEN** |
| `bun run audit:memory-links` | **ALL GREEN** — passed 9, failed 0 |
| `bun run audit:side-chat-privacy` | **ALL GREEN** (unit invariants; no live `/side` turn this run) |

Live instance still shows location duplicate nodes despite green structural suite (same seam as runs a→c).

---

## Chronicle Surfaces (§5)

All hit with bearer token on `6a2bfafd30b7d5f1412cbeff`:

| Surface | OK? | Note |
|---|---|---|
| Recap | ✅ | `current_place: the Rusted Bolt`, `when: a day` |
| Events | ✅ | 20 main events, contiguous seq 1–20 |
| Memories | ✅ | 14 atoms; Mara search returns 4; Mira search returns `[]` |
| Calendar | ✅ | Gregorian months, day 3 after two `/continue day` |
| Threads | ✅ | 2 open threads returned |
| Relationships (Bonds) | ✅ | 6 NPC cards; no player self-name card |
| Locations (Places) | ❌ | 13 nodes — dedup failure (below) |
| Side-chats | ⬛ | Not exercised (GM lane; no `/side` this run) |

---

## Corruption-Class / P1 Bugs

### [SEV: HIGH] Cluster D — Location duplicate nodes (dedup not landing live)
- **World/instance:** GM noir "Neon Divide" iid=`6a2bfafd30b7d5f1412cbeff`
- **Repro:** 19-turn movement loop (agent-chat log `/tmp/agent1-play-6a2bfafd.log`)
- **Expected vs got:** One entity per canonical place. Got 13 nodes with article/parent/sibling duplicates and zero-event orphans.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/locations/6a2bfafd30b7d5f1412cbeff — duplicate / orphan highlights
  [
    {"name":"the Rusted Bolt","event_count":10,"entity_id":"6a2bfb465f863f8b449c071e"},
    {"name":"The Rusted Bolt","event_count":0,"entity_id":"6a2bfb785f863f8b449c0749"},
    {"name":"Rusted Bolt bar","event_count":0,"entity_id":"6a2bfb495f863f8b449c0726"},
    {"name":"old-town","event_count":1,"entity_id":"6a2bfb465f863f8b449c071d","parent_id":null},
    {"name":"old-town","event_count":1,"entity_id":"6a2bfbb85f863f8b449c077b","parent_id":"6a2bfb465f863f8b449c071d"},
    {"name":"The Zenith District","event_count":2,"entity_id":"6a2bfc3e5f863f8b449c07fb"},
    {"name":"The Zenith District","event_count":0,"entity_id":"6a2bfc425f863f8b449c0802"},
    {"name":"the docks","event_count":1,"entity_id":"6a2bfca35f863f8b449c084c"},
    {"name":"docks","event_count":0,"entity_id":"6a2bfc855f863f8b449c0840"},
    {"name":"the Divide","event_count":0,"entity_id":"6a2bfb205f863f8b449c06dd"},
    {"name":"the Divide","event_count":1,"entity_id":"6a2bfc285f863f8b449c07e4","parent_id":"6a2bfb465f863f8b449c071d"}
  ]
  ```
- **Verified how:** `GET /chronicle/locations/:iid`, Mongo `entities` (`type: location`)
- **Known gap?** Cluster D — location fragmentation (matrix row D, fix queue #3)

### [SEV: MED] Cluster D-cursor — Narrated exit, cursor stuck inside venue (seq 8)
- **World/instance:** iid=`6a2bfafd30b7d5f1412cbeff`
- **Repro:** `"I step outside and check where I am after the night passed."` (seq 8, after `/continue day` seq 7)
- **Expected vs got:** Cursor should follow narrated viewpoint to street/outside. Got `location_anchor.name` still `"the Rusted Bolt"` while prose places player on sidewalk outside.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/events/6a2bfafd30b7d5f1412cbeff — seq 8
  {
    "sequence": 8,
    "location_anchor": {"name": "the Rusted Bolt", "name_normalized": "rusted bolt"},
    "travel": null,
    "data.ai_response": "*The heavy steel door groans... He stands on the cracked sidewalk outside the Rusted Bolt, the neon sign above him humming a dying tune in the daylight.*"
  }
  ```
  ```
  // agent-chat log
  [seq 8] location: the Rusted Bolt   (narrative: "stands on the cracked sidewalk outside the Rusted Bolt")
  ```
- **Verified how:** agent-chat log + `GET /chronicle/events/:iid` + Mongo `events`
- **Known gap?** Cluster D-cursor (matrix row D-cursor)

### [SEV: MED] Cluster D-cursor — `/continue day` moves to docks; next turn at bar without travel
- **World/instance:** iid=`6a2bfafd30b7d5f1412cbeff`
- **Repro:** `/continue day` (seq 18) → narrative ends at docks. Next turn `"I ask around the bar..."` (seq 19) → cursor at Rusted Bolt with no travel object on event.
- **Expected vs got:** Either stay at docks until explicit travel, or record travel docks→bar. Got snap to bar (`location_anchor: the Rusted Bolt`) with `travel: null` in Mongo/API.
- **Evidence (RAW):**
  ```json
  // seq 18 → 19 location anchors
  {"sequence":18,"location_anchor":{"name":"the docks"},"time_advanced":"a day"}
  {"sequence":19,"location_anchor":{"name":"the Rusted Bolt"},"travel":null}
  ```
- **Verified how:** agent-chat log + Mongo `events` location_anchor chain
- **Known gap?** Cluster D-cursor

### [SEV: MED] Cluster B1 — Memory invents player surname + conflates characters
- **World/instance:** iid=`6a2bfafd30b7d5f1412cbeff`
- **Repro:** Player intro as `"Kade"` (no surname). Sister `"Vesper"` separate from fixer `"Mara Chen"`.
- **Expected vs got:** Player memories use template name only; sister and fixer stay distinct. Got fabricated `"Kade Chen"` and `"Mara Chen's codename is Vesper"`.
- **Evidence (RAW):**
  ```json
  // GET /chronicle/memories/6a2bfafd30b7d5f1412cbeff?q=mara — active lore atoms
  {"_id":"6a2bfc085f863f8b449c07c6","text":"Kade Chen recalled the name of his fixer contact, Mara Chen...","status":"active","subjects":["Kade Chen"]}
  {"_id":"6a2bfc855f863f8b449c0841","text":"Mara Chen's codename is Vesper, and she runs a safehouse near the docks...","status":"active","subjects":["Mara Chen"],"objects":["safehouse","docks"]}
  ```
  ```json
  // db.characters — separate cards (codex merge OK, memory conflation NOT OK)
  [{"canonical_name":"Mara Chen"},{"canonical_name":"Vesper","persona":"sister of the Operative"}]
  ```
- **Verified how:** `GET /chronicle/memories/:iid`, Mongo `memories` + `characters`
- **Known gap?** Cluster B1 (matrix row B1 — residual AI-subject / identity drift)

### [SEV: LOW] Cluster C — Phantom presence tag (Courier at old-town)
- **World/instance:** iid=`6a2bfafd30b7d5f1412cbeff`
- **Repro:** seq 9 — Mara Chen approaches in chrome jacket; `present_characters` lists `Courier` instead.
- **Expected vs got:** Present list matches on-scene NPC. Got wrong character id in `present`.
- **Evidence (RAW):**
  ```
  // agent-chat log
  [seq 9] location: old-town  present: Courier
  // (player message named "Mira Chen, my fixer contact")
  ```
- **Verified how:** agent-chat log; Bonds later shows `Mara Chen` card (codex corrected) but seq 9 presence tag wrong
- **Known gap?** Cluster C — presence recall vs co-location

---

## Side-Chat Leak Test

**Not run** — GM lane (`is_sentient:false`); no `/side` turns this session. `audit:side-chat-privacy` script green on unit fixtures only. Per playbook §7, GM-world Threads surfacing would be by-design anyway.

---

## Dev-State Notes

- **Created:** instance `6a2bfafd30b7d5f1412cbeff` on template `6a2bd56fafc85d8941c370d5`
- **Deleted:** nothing
- **Redis session bust:** not needed (no rewind)
- **Images:** none generated
- **Servers:** assumed running on localhost:3000 (play completed successfully)

---

## Summary for Triage

| Fix-batch-3 item | Verdict |
|---|---|
| B2a player self-intro codex guard | **PASS** |
| R2 codex merge on name correction | **PASS** |
| Location dedup | **FAIL** |
| Cursor lag (/continue + returns) | **FAIL** |
| Travel from≠to | **PASS** |

**Top live findings:** (1) location dedup still broken — 13 nodes incl. triple Rusted Bolt variants; (2) cursor lag — seq 8 outside prose / inside cursor, seq 18→19 bar snap; (3) B2a/R2 fixes **held** — no Kade card, single Mara codex + superseded Mira memory; (4) new B1 memory corruption — invented `Kade Chen`, Mara/Vesper conflation.
