# Playtest Findings — 2026-06-12d · Agent 5

**Lane:** Character companion (`kind: character`, `is_sentient: true`, `is_nsfw_capable: true`)  
**Genre:** Modern romance / slice-of-life  
**Template ID:** `6a2bd56fafc85d8941c370d6`  
**Fresh instance ID:** `6a2bfb1230b7d5f1412cbf1f` (created this run)  
**World:** “Lena” — warm-but-guarded barista at Paper Moon café  
**Throwaway:** No — persist instance  
**Turns played:** 18 main turns (+ 1 opening event = seq 19)  
**Date:** 2026-06-12

---

## Regression Checks (fix-batch 3 re-verify)

| # | Check | Result | Notes |
|---|---|---|---|
| **(B2a)** | Player-card guard — no card for player self-name | **FAIL** | Swapnil card still minted at seq 6; visible in Bonds with meters null. Fix batch 3 did **not** land for this lane. |
| **(B2b)** | Absent relatives Mira/Mara NOT carded | **FAIL** | Mira sister card minted at seq 9 (`role: side character`). Regression vs 2026-06-12c PASS. |
| **(B1)** | Identity attribution — player facts on player, not companion | **PARTIAL** | Sister Mara→Mira facts on `The Player` entity; no Lena-as-sister inversion. Residual: player↔Swapnil treated as distinct entities; inverted third-person phrasing persists. |
| **(E)** | Explicit-correction supersession — Mara retired + linked | **PASS** | Mara atom superseded at seq 5; forward `updates_memory_ids` chain intact. |
| **(L)** | Bonds shows companion with meters | **PASS** | Lena protagonist with trust 58, affection 51. |
| **(I)** | Event-edit wrong field → 400 | **PASS** | Wrong key `narrative` → 400. Unchanged exact-copy not re-tested (accidental edit during probe mutated seq 3). |
| **(D)** | Location cursor moves on travel | **FAIL** | Single place `the counter` for all 18 post-opening events despite sidewalk/apartment/park travel. |
| **(C)** | Presence — recall ≠ co-location | **PARTIAL FAIL** | Swapnil absent from `present_characters` seq 11–16, 18–19 while player is in-scene at Paper Moon. Mira never present (good). |

---

## Fix-Batch 3 Delta vs 2026-06-12c (Agent 5)

| Cluster | Prior (c) | This run (d) |
|---|---|---|
| **B2a** player card | FAIL — Swapnil carded | **STILL FAIL** — Swapnil carded (unchanged) |
| **B2b** relative carding | PASS — no sister cards | **REGRESSED** — Mira Bonds card |
| **B1** identity | IMPROVED / partial | **STILL PARTIAL** — player/Swapnil split + awkward phrasing |
| **E** supersession | PASS | **PASS** — held |
| **D** location freeze | FAIL | **STILL FAIL** — frozen at `the counter` |
| **C** presence | PASS | **PARTIAL REGRESSION** — player drops from present mid-scene |

---

## Corruption-Class / P0-P1 Findings

### [SEV: HIGH] B2a — Player self-name still minted as Bonds card (fix batch 3 NOT fixed)
- **World/instance:** Character “Lena” iid=`6a2bfb1230b7d5f1412cbf1f`
- **Repro:** Turn 1: `"Hi Lena. I'm Swapnil — just moved here last month."` → codex delta at seq 6 mints Swapnil card.
- **Expected vs got:** Codex rule: *“NEVER create a card for the player.”* Got Swapnil side card in Bonds.
- **Evidence (RAW):**
```json
GET /chronicle/relationships/6a2bfb1230b7d5f1412cbf1f →
{"characters":[
  {"id":"6a2bfb1630b7d5f1412cbf23","name":"Lena","role":"protagonist","meters":{"trust":58,"affection":51}},
  {"id":"6a2bfb7a5f863f8b449c074b","name":"Swapnil","role":"side character","meters":null,"mention_count":2},
  {"id":"6a2bfba85f863f8b449c076e","name":"Mira","role":"side character","meters":null,"mention_count":1}
]}
```
```javascript
// db.characters.find({instance_id: ObjectId("6a2bfb1230b7d5f1412cbf1f")})
// → Lena (protagonist), Swapnil (side character), Mira (side character)
// db.entities: "The Player" (6a2bfb315f863f8b449c06fb) AND "Swapnil" (6a2bfb325f863f8b449c0704) — dual player entities
```
- **Verified how:** Chronicle relationships API + Mongo `db.characters` + `db.entities`.
- **Known gap?** Cluster B2a — **still open**; primary fix-batch-3 target **not resolved**.

### [SEV: MED] B2b — Absent sister Mira carded (regression)
- **World/instance:** Character “Lena” iid=`6a2bfb1230b7d5f1412cbf1f`
- **Repro:** Seq 4 Mara mention, seq 5 correction to Mira → codex delta at seq 9 mints Mira card.
- **Expected vs got:** Sister never on-scene; should be entity-only. Got full Bonds card.
- **Evidence (RAW):** Same relationships JSON above — Mira listed with `role: "side character"`. `dump-seq` seq 9 codex_deltas: `[{"name":"Mira","role":"side character"}]`.
- **Verified how:** Relationships API + `dump-seq.ts` codex_deltas.
- **Known gap?** Cluster B2b — **regressed** from c-run PASS.

### [SEV: MED] B1 — Player/Swapnil entity split + inverted memory phrasing (partial)
- **World/instance:** Character “Lena” iid=`6a2bfb1230b7d5f1412cbf1f`
- **Repro:** Self-intro + identity clarification turns (seq 2, 17).
- **Expected vs got:** Player self-facts attributed to single player entity; companion not conflated. Got dual entities and third-person inversions.
- **Evidence (RAW):**
```javascript
// Active memories (Mongo) — subject_entity_ids on "The Player" (6a2bfb315f863f8b449c06fb):
{"_id":"6a2bfb325f863f8b449c0705","text":"the player welcomed Swapnil to the neighborhood..."}
{"_id":"6a2bfbc75f863f8b449c078f","text":"the player admitted that she does not remember the player's name because the player never told her..."}
{"_id":"6a2bfc215f863f8b449c07da","text":"the player finally understands that Swapnil is not Mira, but her brother..."}
// B1 keystone scan — zero active memories with Lena subject + sister/Mira/Mara text:
db.memories.find({instance_id, status:"active", subject_entity_ids: ObjectId("6a2bfb315f863f8b449c06fa"), text: /sister|Mira|Mara/i}) → []
```
- **Verified how:** Mongo `db.memories` + entity map.
- **Known gap?** Cluster B1 — **partial**; sister facts not on Lena, but player↔Swapnil split is corruption-adjacent.

### [SEV: HIGH] D — Location frozen despite narrated travel (still open)
- **World/instance:** Character “Lena” iid=`6a2bfb1230b7d5f1412cbf1f`
- **Repro:** Seq 7 sidewalk, seq 8 apartment, seq 18 park, seq 19 return to Paper Moon.
- **Expected vs got:** Cursor should mint/move to sidewalk, apartment, park, Paper Moon. Got single anchor for all events.
- **Evidence (RAW):**
```json
GET /chronicle/locations/6a2bfb1230b7d5f1412cbf1f →
{"current_location":{"entity_id":"6a2bfb2d5f863f8b449c06f4","name":"the counter"},
 "places":[{"name":"the counter","event_count":18,"memory_count":13,"first_seen_sequence":2,"last_seen_sequence":19}]}
```
Agent log: every `[seq N]` line shows `location: the counter` (seq 2–19).
- **Verified how:** Locations API + `/tmp/agent5_lena_d.log`.
- **Known gap?** Cluster D — **still open** (chronic across all runs).

### [SEV: MED] C — Player absent from `present_characters` while in-scene (partial regression)
- **World/instance:** Character “Lena” iid=`6a2bfb1230b7d5f1412cbf1f`
- **Repro:** Seq 11–16 player at Paper Moon asking Lena questions; seq 18–19 return visits.
- **Expected vs got:** Player co-located with Lena should appear in `present_characters`. Got `present: Lena` only (Swapnil dropped).
- **Evidence (RAW):** Agent log seq 11–16, 18–19: `present : Lena` (no Swapnil). Seq 17 after identity clarify: `present : Lena, Swapnil` (brief return).
- **Verified how:** `dump-seq.ts` + agent-chat structured tail.
- **Known gap?** Cluster C — partial; Mira never falsely present (good), but player presence drops.

---

## PASS Highlights (raw proof)

### Supersession (Cluster E) — PASS
```javascript
// Mara atom — superseded at correction turn (seq 5):
{_id: "6a2bfb5b5f863f8b449c0733", status: "superseded", is_archived: true,
 text: "...sister named Mara who lives across town...",
 superseded_by_event_ids: ["6a2bfb685f863f8b449c073b"],
 subject_entity_ids: ["6a2bfb315f863f8b449c06fb"]}  // The Player

// Mira correction atom — superseded later at identity clarify (seq 17):
{_id: "6a2bfb6c5f863f8b449c0741", status: "superseded", is_archived: true,
 updates_memory_ids: ["6a2bfb5b5f863f8b449c0733"],
 superseded_by_event_ids: ["6a2bfc1c5f863f8b449c07d3"]}

// Active forward link:
{_id: "6a2bfc215f863f8b449c07da", status: "active",
 updates_memory_ids: ["6a2bfb6c5f863f8b449c0741", "6a2bfbb35f863f8b449c077a"],
 text: "the player finally understands that Swapnil is not Mira, but her brother..."}
```

### Bonds companion (Cluster L) — PASS
```json
{"id":"6a2bfb1630b7d5f1412cbf23","name":"Lena","role":"protagonist",
 "meters":{"trust":58,"affection":51,"fear":0,"rivalry":0}}
```

### Event-edit wrong key (Cluster I) — PASS
```
PUT /chronicle/event/6a2bfb505f863f8b449c072a {"narrative":"changed"}
→ HTTP 400 {"error":"No editable event fields provided. Use ai_response and/or player_input."}
```
*(Unchanged exact-copy check skipped — probe accidentally recurated seq 3.)*

---

## §5 Chronicle Surfaces + Audits

### Fresh instance `6a2bfb1230b7d5f1412cbf1f`

| Surface | Result |
|---|---|
| `GET /chronicle/recap` | `when: "the next morning"`, `where: "the counter"`; bonds lists Swapnil + Mira (no Lena) |
| `GET /chronicle/events` | 19 events, contiguous seq 1–19 |
| `GET /chronicle/memories` | 17 atoms (14 active + 3 superseded hidden from default list) |
| `GET /chronicle/calendar` | Gregorian months ✅; year 1, day 3 after `/continue day` |
| `GET /chronicle/threads` | 3 open threads (includes B1-inverted phrasing thread) |
| `GET /chronicle/relationships` | Lena + Swapnil + Mira |
| `GET /chronicle/locations` | 1 place — frozen at `the counter` |
| `GET /chronicle/side-chats` | `[]` (no side NPC to test Cluster A) |
| `GET /admin/instances/.../continuity-audit` | **healthy: true**, 8/8 ok |

### Deterministic audits (repo-wide synthetic)

| Audit | Result |
|---|---|
| `audit:location` | ✅ ALL INVARIANTS HELD |
| `audit:movement` | ✅ 45 passed, 0 failed |
| `audit:location-resolution` | ✅ ALL INVARIANTS HELD |
| `audit:memory-links` | ✅ 9 passed, 0 failed |
| `continuity-audit` (fresh) | ✅ 8/8 ok |

---

## Not Testable This Run
- **A (side-chat secret leak):** no second on-scene NPC card → side-chat unreachable.
- **NSFW routing:** no explicit NSFW prompt exercised.

---

## Summary

| Item | Value |
|---|---|
| **Template** | `6a2bd56fafc85d8941c370d6` (Lena) |
| **Fresh instance** | `6a2bfb1230b7d5f1412cbf1f` (18 turns, seq 19) |
| **Fix-batch-3 target (B2a)** | **FAIL** — Swapnil player card still minted |
| **Regression scorecard** | E ✅, L ✅, I (wrong-key) ✅ · B2a ❌, B2b ❌, D ❌, C partial ❌, B1 partial |
| **Still-open chronic** | D (location freeze) |
| **New regression** | B2b Mira card (was PASS in c) |
| **Structural audits** | All green |
| **Worlds deleted** | None |

**Net verdict:** Fix batch 3 did **not** close B2a on the Lena lane — Swapnil is still carded after self-intro, with dual player entities (`The Player` + `Swapnil`). B2b regressed (Mira sister card). Supersession (E) and Bonds companion (L) held. Location freeze (D) unchanged. B1 sister attribution on player entity is OK; player/Swapnil split phrasing remains noisy.

**Log:** `/tmp/agent5_lena_d.log`
