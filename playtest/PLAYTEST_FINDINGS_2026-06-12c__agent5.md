# Playtest Findings — 2026-06-12c · Agent 5

**Lane:** Character companion (`kind: character`, `is_sentient: true`, `is_nsfw_capable: true`)  
**Genre:** Modern romance / slice-of-life  
**Template ID:** `6a2bd56fafc85d8941c370d6`  
**Fresh instance ID:** `6a2bea92626fb837070f2b82` (created this run)  
**Existing instance ID:** `6a2bd589afc85d8941c370fd` (continuity / dangling-link re-check only)  
**World:** “Lena” — warm-but-guarded barista at Paper Moon café  
**Throwaway:** No — persist both instances  
**Turns played (fresh):** 18 main turns (+ 1 opening event = seq 19)  
**Date:** 2026-06-12

---

## Regression Checks (fix-batch re-verify)

| # | Check | Result | Notes |
|---|---|---|---|
| (E) | Explicit-correction supersession — Mara→Mira retired + linked | **PASS** | 2 superseded atoms + active correction atom with forward `updates_memory_ids`. Mongo proof below. |
| (L) | Bonds shows companion with meters | **PASS** | Lena (protagonist) visible with trust 62, affection 57. |
| (B2b) | Absent relatives Mira/Mara NOT carded | **PASS** | Bonds lists only Lena + Swapnil — no sister cards. Mira exists as entity only, no codex card. |
| (B2a) | Player-card guard — no card for player | **FAIL** | Swapnil card minted (`6a2beaa67c5c71f2912fee00`, role: newcomer) after player self-introduced. |
| (I) | Event-edit wrong/unchanged field → 400 | **PASS** | Wrong key + unchanged exact `ai_response` both 400. |
| (N2) | Existing dangling `updates_memory_ids` pruned? | **PASS (FIXED)** | Memory `6a2bd5fe…` refs now resolve; dangling scan = 0. |

---

## Fix-Batch Delta vs 2026-06-12b (Agent 5)

| Cluster | Prior (b) | This run (c) |
|---|---|---|
| **B1** identity inversion | FAIL — player facts on Lena; player renamed “Mira” | **IMPROVED / PARTIAL PASS** — sister facts on player entity; no Lena-sister conflation; no Mira-as-player memories |
| **B2b** relative carding | FAIL — Mira + Mara Bonds cards | **PASS** — no sister cards |
| **E** supersession | FAIL — nothing to supersede | **PASS** — full chain |
| **N2** dangling links (existing) | FAIL — 2 dead refs on `6a2bd5fe` | **PASS** — refs exist, scan clean |
| **C** presence recall = co-location | FAIL — Mira in `present_characters` | **PASS** — sister never present |
| **D** location freeze | FAIL — 1 place, 0 travel | **STILL FAIL** — still 1 place through narrated travel |
| **B2a** player card | PASS (Swapnil not carded in b) | **REGRESSED** — Swapnil carded |

---

## Corruption-Class / P0-P1 Findings

### [SEV: MED] B2a — Player self-name minted as Bonds card (regression)
- **World/instance:** Character “Lena” iid=`6a2bea92626fb837070f2b82`
- **Repro:** Turn 1: `"Hi Lena. I'm Swapnil — just moved here last month."` → play through seq 11+.
- **Expected vs got:** Codex rule: *“NEVER create a card for the player.”* Got a Swapnil side card with meters.
- **Evidence (RAW):**
```json
GET /chronicle/relationships/6a2bea92626fb837070f2b82 →
{"characters":[
  {"id":"6a2bea92626fb837070f2b89","name":"Lena","role":"protagonist","meters":{"trust":62,"affection":57}},
  {"id":"6a2beaa67c5c71f2912fee00","name":"Swapnil","role":"newcomer","meters":{"trust":51,"affection":51},"mention_count":8}
]}
```
- **Verified how:** Chronicle relationships API + `db.characters.find({instance_id})` (2 non-protagonist cards: only Swapnil; no Mira/Mara).
- **Known gap?** Cluster B2a — **regressed** on fresh run (was PASS in 2026-06-12b for this lane when player persona was not explicitly named).

### [SEV: MED] B1 — Residual player-entity awkwardness (not keystone FAIL)
- **World/instance:** Character “Lena” iid=`6a2bea92626fb837070f2b82`
- **Repro:** Sister fact + correction + identity clarification turns (seq 4–8, 17).
- **Expected vs got:** Player self-facts must NOT attribute to Lena/companion. **Keystone inversion fixed** for sister facts (subject = player entity). Remaining noise: duplicate player entities (“The Player” + “Swapnil”) and awkward third-person phrasing (*“the player welcomed Swapnil”*).
- **Evidence (RAW) — sister facts correctly on player entity:**
```javascript
// db.memories.find({instance_id: ObjectId("6a2bea92626fb837070f2b82"), text: /sister|Mira|Mara/i})
{
  id: "6a2beb877c5c71f2912fef30", status: "active",
  text: "the player recognizes Swapnil's identity and acknowledges that Mira is his sister...",
  subject: [ObjectId("6a2beaa67c5c71f2912fedfd"), ObjectId("6a2beaa67c5c71f2912fedfe")],
  updates_memory_ids: ["6a2beade7c5c71f2912fee54","6a2beae77c5c71f2912fee63"]
}
// B1 scan: zero active memories with Lena entity subject + sister/Mira/Mara text
db.memories.find({instance_id, status:"active", subject_entity_ids: ObjectId("6a2beaa67c5c71f2912fee01"), text: /sister|Mira|Mara/i}) → []
```
- **Verified how:** Mongo `db.memories` + entity map (`Lena`=`6a2beaa67c5c71f2912fee01`, `The Player`=`6a2beaa67c5c71f2912fedfd`, `Swapnil`=`6a2beaa67c5c71f2912fedfe`, `Mira` entity=`6a2beae77c5c71f2912fee62` — entity only, no character card).
- **Known gap?** Cluster B1 — **materially improved**; no longer keystone-red on sister attribution. Leftover entity split + phrasing is med/low.

### [SEV: HIGH] D — Location frozen despite narrated travel (still open)
- **World/instance:** Character “Lena” iid=`6a2bea92626fb837070f2b82`
- **Repro:** Seq 9 `"I walk outside the café..."`, seq 10 `"I go to my apartment..."`, seq 12 `"I come back to Paper Moon the next morning."`
- **Expected vs got:** Cursor should move (sidewalk, apartment, Paper Moon). Got single anchor for all 18 post-opening events.
- **Evidence (RAW):**
```json
GET /chronicle/locations/6a2bea92626fb837070f2b82 →
{"current_location":{"entity_id":"6a2beaa17c5c71f2912fedf2","name":"the coffee shop"},
 "places":[{"name":"the coffee shop","event_count":18,"memory_count":18}]}
```
Agent log: every `[seq N]` line shows `location: the coffee shop` (seq 2–19).
- **Verified how:** Locations API + `/tmp/agent5_lena_fresh.log`.
- **Known gap?** Cluster D — **still open**.

---

## Quality / UX (non-corruption)

### [SEV: LOW] Awkward “player ↔ Swapnil” memory phrasing
- **Evidence (RAW):** `db.memories` id `6a2beaa67c5c71f2912fedff`, status active: *“the player welcomed Swapnil to the neighborhood…”* — subject `6a2beaa67c5c71f2912fedfd` (The Player), object `6a2beaa67c5c71f2912fedfe` (Swapnil).
- **Known gap?** Follow-on from B2a player card + dual player entities — not a silent corruption path.

### [SEV: MED] G — Recap bonds omit protagonist (UX)
- **Evidence (RAW):** `GET /chronicle/recap/6a2bea92626fb837070f2b82` → `bonds: [{name:"Swapnil",...}]` only; Lena absent from recap bonds despite being the companion. `when: "tomorrow"` now populated (improved vs b).

---

## PASS Highlights (raw proof)

### Supersession (Cluster E) — PASS
```javascript
// Superseded atoms (Mongo):
{_id: ObjectId("6a2beade7c5c71f2912fee54"), status:"superseded",
 text: "...correction of their sister's name from Mara to Mira...",
 superseded_by_event_ids: [ObjectId("6a2beb7e7c5c71f2912fef24")]}
{_id: ObjectId("6a2beae77c5c71f2912fee63"), status:"superseded",
 text: "...sister, Mira, lives in Capitol Hill...",
 superseded_by_event_ids: [ObjectId("6a2beb7e7c5c71f2912fef24")]}
// Forward link on correction atom:
{_id: ObjectId("6a2beb877c5c71f2912fef30"), status:"active",
 updates_memory_ids: [ObjectId("6a2beade7c5c71f2912fee54"), ObjectId("6a2beae77c5c71f2912fee63")]}
```

### B2b absent-relative guard — PASS
```json
GET /chronicle/relationships/6a2bea92626fb837070f2b82 → characters: ["Lena","Swapnil"] only
db.characters.find({instance_id, name: /Mira|Mara/i}) → null
db.entities.findOne({canonical_name:"Mira"}) → entity exists, no matching character doc
```

### Presence (Cluster C) — PASS
Agent log `present` lines: only `Lena` (seq 2–11, 19) or `Lena, Swapnil` (seq 12–18). **Mira/Mara never in `present_characters`.**

### Existing-instance N2 dangling links — PASS (fixed)
```javascript
db.memories.findOne({_id: ObjectId("6a2bd5fe7a263366b3cfabf7")})
→ updates_memory_ids: ["6a2bd5d37a263366b3cfabb4","6a2bd5e67a263366b3cfabd4"]
→ both refs EXISTS (status active)
// Full scan:
db.memories.find({instance_id: ObjectId("6a2bd589afc85d8941c370fd"), updates_memory_ids: {$ne:[]}})
  .filter(refs missing) → dangling count: 0
```
Prior run (b): both refs MISSING. **N2 closed on existing save.**

---

## §5 Chronicle Surfaces + Audits

### Fresh instance `6a2bea92626fb837070f2b82`
| Surface | Result |
|---|---|
| `GET /chronicle/recap` | `when: "tomorrow"`, `where: "the coffee shop"`, recap_text populated; bonds shows Swapnil only |
| `GET /chronicle/events` | 19 events, contiguous seq 1–19 |
| `GET /chronicle/memories` | 20 atoms (18 active + 2 superseded hidden from default list) |
| `GET /chronicle/calendar` | Gregorian months; year 1; day advanced to 3 after `/continue day` |
| `GET /chronicle/threads` | 3 open threads |
| `GET /chronicle/relationships` | Lena + Swapnil (no sister cards) |
| `GET /chronicle/locations` | 1 place — frozen |
| `GET /chronicle/side-chats` | `[]` (no side NPC to test Cluster A) |

### Existing instance `6a2bd589afc85d8941c370fd` (re-check only)
| Check | Result |
|---|---|
| `GET /admin/instances/.../continuity-audit` | **healthy: true**, 8/8 ok |
| Memory `6a2bd5fe` dangling refs | **FIXED** — refs resolve |
| Bonds | Still shows legacy Mira + Mara sister cards from prior run (historical data, not re-played) |

### Deterministic audits (repo-wide synthetic)
| Audit | Result |
|---|---|
| `audit:location` | ✅ ALL INVARIANTS HELD |
| `audit:movement` | ✅ 45 passed, 0 failed |
| `audit:location-resolution` | ✅ ALL INVARIANTS HELD |
| `audit:memory-links` | ✅ 9 passed, 0 failed |
| `scripts/rewind-audit.ts` | ✅ clean |
| `continuity-audit` (fresh) | ✅ 8/8 ok |
| `continuity-audit` (existing) | ✅ 8/8 ok |

### Mutation regression (fresh)
- `PUT /chronicle/event/6a2beaa17c5c71f2912fedf3` `{"narrative":"..."}` → **400** `"No editable event fields provided."`
- Same event, exact unchanged `data.ai_response` → **400** `"Event edit did not change ai_response or player_input."`

---

## Not Testable This Run
- **A (side-chat secret leak):** no second NPC card → side-chat unreachable.
- **NSFW routing:** no explicit NSFW prompt; model `google/gemma-4-31b-it` throughout log.

---

## Summary

| Item | Value |
|---|---|
| **Template** | `6a2bd56fafc85d8941c370d6` (Lena) |
| **Fresh instance** | `6a2bea92626fb837070f2b82` (18 turns, seq 19) |
| **Existing instance** | `6a2bd589afc85d8941c370fd` (audit + N2 re-check only) |
| **Regression scorecard** | E ✅, L ✅, B2b ✅, I ✅, N2 ✅ · B2a ❌ |
| **B1** | Improved — sister facts on player entity; no Mira-as-player inversion |
| **Still-open** | D (location freeze), B2a (Swapnil card) |
| **Structural audits** | All green (including continuity on both instances) |
| **Worlds deleted** | None |

**Net verdict:** Major forward movement on the Lena lane keystone (B1 sister attribution, B2b relative carding, E supersession, C presence, N2 dangling links). Location freeze (D) unchanged. New/regressed issue: player self-name “Swapnil” minted as a Bonds card (B2a).

**Log:** `/tmp/agent5_lena_fresh.log`
