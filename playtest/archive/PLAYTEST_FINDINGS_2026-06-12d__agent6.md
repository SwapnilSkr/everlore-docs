# Playtest Findings — 2026-06-12d — Agent 6 (fix batch 3 re-test)

**Lane:** CHARACTER companion (kind:`character`, is_sentient:`true`, is_nsfw_capable:`false`)  
**Genre:** Fantasy adventurer ("Elara the Ironbark Ranger")  
**Template:** `6a2bd570afc85d8941c370d7`  
**Fresh instance:** `6a2bfb1b30b7d5f1412cbf29` (created this run)  
**Turns driven:** 18 (seq 1–18)

---

## Fix Batch 3 — Primary Verification

| Check | Result | Evidence |
|---|---|---|
| **No Kael player codex card** (B2a) | **PASS** | `GET /chronicle/relationships/6a2bfb1b30b7d5f1412cbf29` → only Elara + Merchant; Mongo `db.characters.find({instance_id})` → `Elara`, `Merchant` only — **no Kael card** despite self-intro at seq 4 |
| **No Merchant passer-by card** | **FAIL** | Same Bonds response mints `Merchant` (`role:"non-player character"`, `mention_count:1`) after vague passer-by at seq 14 |
| **B1 shiver — player sensation not attributed to Elara** | **FAIL** | Player seq 11: `"I feel a cold shiver when I see clawed prints…"` → memory subject `Elara` (see below) |
| **Location dedup — "the X" vs "X"** | **FAIL** | `the thornwood` + `thornwood` both exist as location entities; Places API lists two `the Thornwood` nodes |

---

## Regression Checks (§4 — still holding)

| # | Check | Result | Evidence |
|---|---|---|---|
| (a) | Wrong field edit → HTTP 400 | **PASS** | `PUT /chronicle/event/6a2bfb645f863f8b449c0737` `{"narrative":"changed"}` → `{"error":"No editable event fields provided..."}` HTTP **400** |
| (b) | Empty `ai_response` edit → HTTP 400 | **PASS** | `{"ai_response":""}` → `{"error":"ai_response cannot be empty."}` HTTP **400** |
| (c) | Unchanged `ai_response` edit → HTTP 400 | **PASS** | Copied exact `.data.ai_response` from seq 4 → `{"error":"Event edit did not change ai_response or player_input."}` HTTP **400** |
| (d) | Bonds shows companion | **PASS** | Elara protagonist `trust:55`, `affection:50`, `mention_count:18` |
| (e) | Absent-relative sisters not carded | **PASS** | Bonds has no Mira/Mara cards; entities exist (`mira`, `player's sister`) but no codex cards |
| (f) | Recap `when` non-null | **PASS** | `"when":"a day"`, `"current_place":"the Thornwood"` |

---

## Fix-Batch Delta vs 2026-06-12c (Agent 6 lane)

| Finding (c run) | d run verdict |
|---|---|
| Kael player card minted (B2a regression) | **CLOSED** — no Kael card after self-intro |
| Merchant passer-by carded | **STILL FAIL** — Merchant card minted again |
| B1 Elara-subject shiver inversion | **STILL FAIL** — identical mis-attribution pattern |
| Location article duplicates (`eastern ridge`×2, `Thornwood`×2) | **PARTIAL** — `eastern ridge` now singular; **`the thornwood` / `thornwood` split persists** |
| Cursor lag on "return to camp" | **IMPROVED** — seq 15–16 cursor moved to `the Thornwood`; orphan `camp` entity minted with 0 events |

---

## Corruption-Class Bugs

### [SEV: HIGH] B1 — Residual identity attribution inversion (player sensation → Elara)
- **World/instance:** character "Elara the Ironbark Ranger" iid=`6a2bfb1b30b7d5f1412cbf29`
- **Repro:**
  1. Player: `"I feel a cold shiver when I see clawed prints near the eastern ridge."` (seq 11)
  2. `GET /chronicle/memories/:iid?q=shiver`
- **Expected vs got:**
  - **Expected:** Memory subject = player/Kael (player felt the shiver).
  - **Got:** `"Elara felt a cold shiver when she saw clawed prints near the eastern ridge..."` with `subjects:["Elara"]`.
- **Evidence (RAW):**
```json
{
  "_id": "6a2bfc275f863f8b449c07e1",
  "text": "Elara felt a cold shiver when she saw clawed prints near the eastern ridge, recognizing that something dangerous was hunting in the area.",
  "subjects": ["Elara"],
  "source_event_ids": ["6a2bfc225f863f8b449c07db"],
  "sequence": 11
}
```
- **Also in recap `open_threads`:**
```json
{"text":"Elara felt a cold shiver when she saw clawed prints near the eastern ridge, recognizing that something dangerous was hun","importance":4}
```
- **Known gap?** Cluster B1 — **fix batch 3 did not close keystone**.

### [SEV: HIGH] B1-variant — Recap thread inverts player/Kael identity
- **Repro:** After player self-intro as Kael (seq 4), check recap open threads.
- **Expected:** Player facts reference the player as Kael, not as a third party.
- **Got:** `"the player feels a mixture of suspicion and curiosity towards Kael, the traveler from the eastern villages..."`
- **Evidence (RAW):** `GET /chronicle/recap/6a2bfb1b30b7d5f1412cbf29` → `open_threads[4]`
- **Known gap?** Cluster B1 — NEW evidence this run; player and Kael treated as distinct entities in curation.

### [SEV: MED] B2 — Passer-by merchant carded (guard still open)
- **Repro:** `"A merchant passes by on the forest road. He looks tired."` (seq 14)
- **Expected:** Vague passer-by should NOT mint a codex card.
- **Got:** Full Bonds card for `Merchant`.
- **Evidence (RAW):**
```json
{
  "id": "6a2bfc575f863f8b449c0812",
  "name": "Merchant",
  "role": "non-player character",
  "disposition": "unknown",
  "mention_count": 1,
  "meters": null
}
```
- **Mongo confirm:** `db.characters.find({instance_id})` → `Merchant` with `mention_count:1`
- **Known gap?** Passer-by carding guard — **not fixed in batch 3**.

### [SEV: MED] D — Location duplicate nodes ("the Thornwood" vs "Thornwood")
- **Repro:** 18 turns with movement through Thornwood / Ironbark forest / eastern ridge.
- **Expected:** Article-normalized dedup — one node per place.
- **Got:** Two location entities + two Places API entries for Thornwood:
```json
{"places":[
  {"entity_id":"6a2bfc5e5f863f8b449c081b","name":"the Thornwood","parent_id":"6a2bfb315f863f8b449c06fc","event_count":4},
  {"entity_id":"6a2bfbf95f863f8b449c07ad","name":"the Thornwood","parent_id":null,"event_count":0}
]}
```
- **Mongo entity split (RAW):**
```
9c07ad location "the thornwood"
9c081b location "thornwood"
9c06fc location "ironbark forest"
9c0822 location "camp" (0 events — orphan from "return to camp")
9c07e0 location "eastern ridge" (singular — improvement vs c run)
```
- **Known gap?** Cluster D — **still open**; article variant unfixed.

---

## Positive Deltas (fix batch 3)

### [SEV: info] B2a — Player self-intro no longer cards Kael
- **Repro:** `"My name is Kael. I'm a traveler from the eastern villages."` (seq 4)
- **Expected:** No player codex card in character world.
- **Got:** Bonds = `[Elara, Merchant]` only; `[codex] updated` at seq 4 updated Elara, not a new player card.
- **Evidence:** `GET /chronicle/relationships/6a2bfb1b30b7d5f1412cbf29` — no Kael entry; Mongo characters collection confirms.
- **Verdict:** **CLOSED vs c run** — primary batch-3 target landed.

### [SEV: info] B2b — Absent-relative guard held
- Sister Mira mentioned at seq 7/13; no Mira/Mara Bonds cards (entities `mira`, `player's sister` exist at graph level only).

### [SEV: info] D-cursor — Return-to-camp cursor improved
- Seq 15 player: `"I return to the camp by the Thornwood edge."` → `location_anchor: the Thornwood` (seq 15–18). Continuity-audit: `"Current place: the Thornwood"`. Orphan `camp` entity still minted (0 events).

---

## Continuity-Audit

`GET /admin/instances/6a2bfb1b30b7d5f1412cbf29/continuity-audit`:

```json
{
  "healthy": true,
  "summary": {"ok": 8, "warn": 0, "fail": 0},
  "maxSequence": 18,
  "checks": [
    {"name": "event_sequence_integrity", "status": "ok", "detail": "18 events, sequences contiguous."},
    {"name": "single_protagonist", "status": "ok", "detail": "Exactly one protagonist card (Elara)."},
    {"name": "memory_entity_refs", "status": "ok", "detail": "19 active memories; all entity refs resolve."},
    {"name": "location_cursor", "status": "ok", "detail": "Current place: the Thornwood."}
  ]
}
```

**8/8 PASS**

---

## §5 Audit Results

| Audit | Status | Detail |
|---|---|---|
| `audit:location` | ✅ PASS | ALL INVARIANTS HELD |
| `audit:movement` | ✅ PASS | 45/45 passed |
| `audit:location-resolution` | ✅ PASS | ALL INVARIANTS HELD |
| `audit:memory-links` | ✅ PASS | 9/9 passed |
| `continuity-audit` (fresh) | ✅ PASS | 8/8 ok |

---

## §5 Chronicle Surface Coverage

| Endpoint | Result |
|---|---|
| `GET /chronicle/recap/:iid` | ✅ `when:"a day"`, B1 shiver in open_threads |
| `GET /chronicle/events/:iid` | ✅ 18 events; edit regression exercised |
| `GET /chronicle/memories/:iid?q=shiver` | ✅ B1 FAIL evidence |
| `GET /chronicle/calendar/:iid` | ✅ Themed months (Ironbark, Leaffall, Mistveil, …) |
| `GET /chronicle/relationships/:iid` | ✅ B2a PASS (no Kael), B2 FAIL (Merchant) |
| `GET /chronicle/locations/:iid` | ✅ D duplicate Thornwood nodes |
| `PUT /chronicle/event/:eventId` | ✅ wrong/empty/unchanged → 400 |
| `GET /admin/instances/:iid/continuity-audit` | ✅ 8/8 |

---

## Summary

- **Total turns:** 18 on fresh instance `6a2bfb1b30b7d5f1412cbf29`
- **Fix batch 3 verdict:**
  - **PASS:** Kael player card guard (B2a closed vs c run)
  - **FAIL:** Merchant passer-by card; B1 shiver inversion; location `"the X"` / `"X"` dedup
- **Regression checks:** 6/6 PASS (edit guards, Bonds companion, absent-relative sisters, recap `when`)
- **Still open:** B1 (shiver + player/Kael recap inversion), Merchant carding, D (Thornwood split)
- **Deterministic audits:** All green

**Worlds left in place:**
- Template `6a2bd570afc85d8941c370d7` (persisted)
- Fresh instance `6a2bfb1b30b7d5f1412cbf29` (persisted, NOT deleted)

**Dev-state mutations:** Created 1 new instance. No rate-limit resets. No rewind performed. No event edits committed (unchanged-copy test only).
