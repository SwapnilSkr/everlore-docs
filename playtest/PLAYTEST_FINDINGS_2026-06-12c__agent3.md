# Agent-3 Playtest Findings — 2026-06-12c (re-test after fix batch)

**Lane:** Sentient world (kind:`world`, is_sentient:`true`) — modern city-AI **Meridian City**  
**Template ID:** `6a2bd564afc85d8941c370d2`  
**Fresh instance ID:** `6a2bea91626fb837070f2b81` (created this run)  
**Existing instance ID:** `6a2bd56eafc85d8941c370d4` (continuity / rewind check only)

**Total turns driven (fresh):** 20 — 18 main-story (seq 2–18) + 2 side-chat (seq 12–13 with Mira).  
**Audits run:** `continuity-audit` (both instances 8/8 ok), `audit:location`, `audit:memory-links` — all green.

---

## Regression Checks (verify live)

| Check | Status | Evidence |
|---|---|---|
| **B1 — badge L-4472 attributes to PLAYER not AI** | **PASS** | Fresh instance: Mongo memory `6a2bec0c7c5c71f2912fef91` (seq 18): `subjects:["player"]`, `subject_entity_ids → entity canonical_name "The Player"` (`type:"player"`). NOT The City. API `GET /chronicle/memories/6a2bea91626fb837070f2b81?q=badge` returns same atom. |
| **N5 — sentient AI always in `present_characters`** | **PASS** | All 18 main turns show `present : The City` (batch1 seq 2–11, batch2 seq 14–18). No empty-present turns. |
| **Player-card guard (B2a)** | **FAIL** | `GET /chronicle/relationships/6a2bea91626fb837070f2b81` minted an **Alex** codex card (`6a2beb4a7c5c71f2912feee9`, role `transit engineer`) after player said `"My name is Alex"`. A separate player entity (`The Player`, `type:player`) also exists — dual representation. |
| **Bonds — AI protagonist visible with meters** | **PASS** | Bonds shows *The City* protagonist with meters (`trust:55`, `affection:51`). |
| **(I) Event-edit wrong field → 400** | **PASS** | `PUT /chronicle/event/6a2bec087c5c71f2912fef8d` with `{"narrative":"..."}` → HTTP **400** `{"error":"No editable event fields provided. Use ai_response and/or player_input."}` |
| **(E) Explicit-correction supersession (Mara→Mira)** | **PASS** | Mongo: seq-6 Mara memory `status:"superseded"`, `superseded_by_event_ids:[ObjectId('6a2beb087c5c71f2912fee99')]`. Seq-7 correction has `updates_memory_ids` populated. Sister card canonical name **Mira** on fresh instance. |
| **A — side-chat secret leak (sentient, protagonist NOT knower)** | **PASS** | Side-chat secret: *stole backup rain-sync keycard from vault*. Mongo side_chat atoms (`6a2bebb17c5c71f2912fef56`, `6a2bebb47c5c71f2912fef59`) have `known_by: ["The Player (player)", "Mira (character)"]` — **The City NOT in known_by**. Echoes search `q=keycard|stole|vault` → `[]`. Threads/recap contain no keycard text. Main turn seq 14 prose does not reference the secret. |
| **N3 — existing instance sister card stale name after rewind** | **STILL OPEN** | Existing iid `6a2bd56eafc85d8941c370d4`: Mongo char `6a2bd8bfafc85d8941c37114` → `canonical_name:"Mara"`, `aliases:["Mara","Mira"]`. API Bonds: `"name":"Mara"`. Rewind re-mint did NOT adopt corrected canonical **Mira**. |
| **Existing continuity-audit** | **PASS** | `GET /admin/instances/6a2bd56eafc85d8941c370d4/continuity-audit` → `healthy:true`, 8/8 ok. |

---

## B1 — Identity Attribution (PRIMARY VERIFY)

### [SEV: HIGH → CLOSED on fresh instance] Badge L-4472 now attributes to player entity

- **World/instance:** sentient "Meridian City" iid=`6a2bea91626fb837070f2b81`
- **Repro:** Player stated `"My badge is still L-4472. I am a transit engineer."` (seq 18).
- **Expected vs got:** Badge belongs to the **player persona**, not The City AI protagonist. **Got:** memory subjects = `player`, entity = `The Player` (type `player`).
- **Evidence (RAW — Mongo):**
```json
{
  "_id": "6a2bec0c7c5c71f2912fef91",
  "text": "the player recognizes the player's badge L-4472, affirming their identity as a transit engineer and acknowledging their role as a healer of the iron veins.",
  "subjects": ["player"],
  "subject_entity_ids": ["6a2beaa97c5c71f2912fee07"],
  "subject_entity": { "canonical_name": "The Player", "type": "player" }
}
```
- **Evidence (RAW — API):** `GET /chronicle/memories/6a2bea91626fb837070f2b81?q=badge` → same atom, `subjects:["player"]`.
- **Verified how:** Mongo `db.memories` + entity lookup `db.entities.findOne({_id: ObjectId("6a2beaa97c5c71f2912fee07")})`.
- **Known gap?** Cluster B1 — **FIX LANDED on fresh play.** Existing instance `6a2bd56eafc85d8941c370d4` still carries pre-fix atoms (subjects `["The City"]` for badge memories) — stale data, not re-played.

**Note:** Seq-3 intro memory also uses `subjects:["player"]` / `The Player` entity (not The City). Minor residual: seq-16 memory subjects `["Alex"]` (codex card entity) instead of `player` — see B2 variant below.

---

## Side-Chat Leak Test (Cluster A — sentient lane)

### [SEV: HIGH] PASS — no leak to main surfaces where protagonist is not a knower

- **World/instance:** sentient "Meridian City" iid=`6a2bea91626fb837070f2b81`
- **Repro:**
  1. `/side 6a2beaf57c5c71f2912fee79` — *"I stole a backup rain-sync keycard from the vault last night. Nobody can know."*
  2. `/side 6a2beaf57c5c71f2912fee79` — *"Promise me you won't tell the City or anyone else about the keycard."*
  3. Main turn seq 14: *"I return to the Plaza. City, any updates on sector 4?"* — narrative does NOT mention keycard/vault/theft.
- **Expected vs got:** Secret must NOT surface in main Echoes/Threads/Recap/narration because The City (protagonist) ∉ `known_by_entity_ids`.
- **Evidence (RAW — Mongo side_chat atoms):**
```json
{
  "_id": "6a2bebb17c5c71f2912fef56",
  "origin": "side_chat",
  "text": "Mira agreed to keep the player's secret about stealing a backup rain-sync keycard from the vault...",
  "known_by_entity_ids": ["6a2beaa97c5c71f2912fee07", "6a2beaf57c5c71f2912fee7a"],
  "known_by_resolved": ["The Player (player)", "Mira (character)"],
  "city_protagonist_in_known": false
}
```
- **Evidence (RAW — API searches):**
  - `GET /chronicle/memories/6a2bea91626fb837070f2b81?q=keycard` → `[]`
  - `GET /chronicle/memories/6a2bea91626fb837070f2b81?q=stole` → `[]`
  - `GET /chronicle/memories/6a2bea91626fb837070f2b81?q=vault` → `[]`
  - `GET /chronicle/threads/6a2bea91626fb837070f2b81` — no keycard/vault/stole in open threads
  - `GET /chronicle/recap/6a2bea91626fb837070f2b81` — recap mentions badge recognition (seq 18), not keycard
- **Verified how:** Mongo `origin:"side_chat"` docs + protagonist entity_id cross-check + Chronicle API searches.
- **Known gap?** Cluster A / N6 — **gate appears fixed for this lane.** Echoes search no longer surfaces side_chat atoms to main view.

---

## Existing-Instance Check — N3 Sister Card Stale Name

### [SEV: MED] STILL OPEN — rewind re-mint keeps canonical `"Mara"` not `"Mira"`

- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4` (prior run, not modified this session)
- **Repro:** Prior agent corrected `"Mara→Mira"`, rewound, re-played. Card re-minted with stale canonical.
- **Expected vs got:** Canonical name should be **Mira** after correction propagated through rewind repair.
- **Evidence (RAW — Mongo):**
```json
{
  "_id": "6a2bd8bfafc85d8941c37114",
  "canonical_name": "Mara",
  "aliases": ["Mara", "Mira"],
  "role": "sister",
  "created_at": "2026-06-12T10:00:31.565Z"
}
```
- **Evidence (RAW — API):** `GET /chronicle/relationships/6a2bd56eafc85d8941c370d4` → `{"id":"6a2bd8bfafc85d8941c37114","name":"Mara","role":"sister"}`.
- **Contrast (fresh instance):** Sister card correctly minted as **Mira** (`6a2beaf57c5c71f2912fee79`) after inline correction — N3 is rewind-specific, not fresh-play.
- **Verified how:** Mongo `db.characters.findOne` + Bonds API on existing instance (no new turns on that save).
- **Known gap?** Cluster N3 in merged 2026-06-12b — **still open.**

---

## Still-Open / Partial Issues

### [SEV: MED] B2-variant — Alex codex card minted for player self-introduction

- **World/instance:** iid=`6a2bea91626fb837070f2b81`
- **Repro:** Player: `"My name is Alex. I am a transit engineer. My badge is L-4472."` (seq 3).
- **Expected vs got:** No Bonds/codex card for the player persona in sentient worlds. **Got:** Alex card `6a2beb4a7c5c71f2912feee9` with role `transit engineer`, plus separate `The Player` entity for memory attribution.
- **Evidence (RAW — API):**
```json
{"id":"6a2beb4a7c5c71f2912feee9","name":"Alex","role":"transit engineer","meters":{"trust":51,"affection":50}}
```
- **Evidence (RAW — Mongo):** `db.characters.findOne({_id: ObjectId("6a2beb4a7c5c71f2912feee9")})` → `canonical_name:"Alex"`, `is_protagonist:false`.
- **Known gap?** Cluster B2a — **regression vs 2026-06-12b agent3 PASS.** B1 fix introduced player entity but Alex card still mints from self-naming.

### [SEV: MED] D3 — Location cursor lags behind narrated travel

- **World/instance:** iid=`6a2bea91626fb837070f2b81`
- **Repro:** Seq 17 player: `"I head to sector 4 maintenance core to fix the rain-sync."` → `location: the Plaza` (frame dump). Place node `sector 4 maintenance core` exists with `event_count:0`.
- **Expected vs got:** Cursor should move to sector 4 maintenance core on seq 17.
- **Evidence:** `/tmp/agent3_batch2.log` seq 17: `location: the Plaza`. API `GET /chronicle/locations/6a2bea91626fb837070f2b81` → `current_location.name: "the Plaza"`.
- **Known gap?** Cluster D — **still open.**

### [SEV: LOW] D1 — Location graph still fragments (article/parent variance)

- **Evidence (RAW — API places):** `"the city"` + child `"Sector 4"`; separate orphan nodes `"the central tower"`, `"the transit hub"`, `"sector 4 maintenance core"` all with `parent_id:null`, several with `event_count:0`.
- **Known gap?** Cluster D — **still open**, milder than prior run (no article dupes like "the Plaza" vs "Plaza of the Tides" yet).

### [SEV: LOW] C — Alex appears in `present_characters` alongside The City

- **Repro:** Seq 16–18: `present : The City, Mira, Alex` — Alex (player codex card) listed as co-located NPC.
- **Expected vs got:** Player persona should not appear as a present NPC card in sentient worlds.
- **Evidence:** `/tmp/agent3_batch2.log` seq 16–18 present lines.
- **Known gap?** Related to B2-variant + Cluster C.

---

## Held GREEN (no regression)

- Side-chat time freeze + dedicated thread surface (`GET /chronicle/side-chats/:iid` → 1 thread, Mira, 2 turns).
- Wrong-field event edit → 400.
- Correction supersession chain (Mara→Mira) on fresh instance.
- Calendar `month_names` populated (Gregorian) on fresh instance.
- `continuity-audit` 8/8 on both fresh and existing instances.
- `audit:location` ALL INVARIANTS HELD; `audit:memory-links` 9/9 passed.

---

## §5 Chronicle Endpoint Coverage (fresh instance)

| Endpoint | Status |
|---|---|
| `GET /chronicle/recap/:iid` | ✅ `when:"a day"`, `current_place:"the Plaza"`, recap_text populated |
| `GET /chronicle/events/:iid` | ✅ 18 events |
| `GET /chronicle/memories/:iid` | ✅ 20 memories; badge search works |
| `GET /chronicle/calendar/:iid` | ✅ Gregorian month_names |
| `GET /chronicle/threads/:iid` | ✅ 6 open, no side-chat leak |
| `GET /chronicle/relationships/:iid` | ✅ The City + Mira + Alex |
| `GET /chronicle/side-chats/:iid` | ✅ 1 thread (Mira) |
| `GET /chronicle/locations/:iid` | ✅ 6 places |
| `PUT /chronicle/event/:id` (wrong key) | ✅ 400 |

---

## Summary

| Area | Verdict |
|---|---|
| **B1 badge attribution** | **FIX LANDED** — `L-4472` → `subjects:["player"]`, entity `The Player`, not The City |
| **Sentient AI in present_characters** | **FIX LANDED** — The City present every main turn |
| **Side-chat leak (sentient, protagonist ∉ knower)** | **PASS** — no leak to Echoes/Threads/Recap/main prose |
| **Player-card guard** | **FAIL** — Alex codex card minted despite player entity existing |
| **N3 rewind sister stale name (existing)** | **STILL OPEN** — canonical `"Mara"` persists |
| **Location cursor / fragmentation** | **STILL OPEN** — D3 lag + D1 orphan nodes |

**Worlds left in place:** Template `6a2bd564afc85d8941c370d2` + fresh instance `6a2bea91626fb837070f2b81` (seq 18) + existing instance `6a2bd56eafc85d8941c370d4` (unchanged, seq 32).

**Logs:** `/tmp/agent3_batch1.log`, `/tmp/agent3_batch2.log`
