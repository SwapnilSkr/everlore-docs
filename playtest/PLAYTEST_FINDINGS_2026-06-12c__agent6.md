# Playtest Findings — 2026-06-12c — Agent 6

**Lane:** CHARACTER companion (kind:`character`, is_sentient:`true`, is_nsfw_capable:`false`)  
**Genre:** Fantasy adventurer ("Elara the Ironbark Ranger")  
**Template:** `6a2bd570afc85d8941c370d7`  
**Fresh instance:** `6a2bea8c626fb837070f2b77` (created this run)  
**Existing instance (continuity-audit):** `6a2bd57eafc85d8941c370f0` (from 2026-06-12b run)  
**Turns driven:** 16 (seq 1–16) on fresh instance.

---

## Regression Checks (§4 — verify live)

| # | Check | Result | Evidence |
|---|---|---|---|
| (a) | **Empty `ai_response` edit → HTTP 400** | **PASS** | `PUT /chronicle/event/6a2beb6d7c5c71f2912fef12` body `{"ai_response":""}` → `{"error":"ai_response cannot be empty."}` HTTP **400** |
| (b) | **Wrong field edit → HTTP 400** | **PASS** | `PUT` body `{"narrative":"changed text"}` → `{"error":"No editable event fields provided. Use ai_response and/or player_input."}` HTTP **400** |
| (c) | **Unchanged `ai_response` edit → HTTP 400** | **PASS** | Copied exact `data.ai_response` from seq 4 (`6a2beab57c5c71f2912fee1a`, len=650) via `GET /chronicle/events/:iid` → `{"error":"Event edit did not change ai_response or player_input."}` HTTP **400** |
| (d) | **Valid changed edit → HTTP 200** | **PASS** | `PUT` body `{"ai_response":"[QA EDIT] The mist parts slightly..."}` → `{"success":true,"recuration_queued":true,...}` HTTP **200** |
| (e) | **Bonds shows companion** | **PASS** | `GET /chronicle/relationships/6a2bea8c626fb837070f2b77` → Elara protagonist with `trust:61`, `affection:53`, `fear:0`, `rivalry:0` |
| (f) | **Absent-relative carding blocked** (Mira/Mara sisters) | **PASS** | Fresh run: Bonds lists Elara + Merchant + Kael only — **no Mira/Mara sister cards** despite sister mentions at seq 4 and seq 10 |
| (g) | **Recap `when` non-null** | **PASS** | Fresh: `"when":"a day"`. Existing: `"when":"a day"` (was `null` in 2026-06-12b run — **fixed**) |

**Note on edit-test methodology:** `GET /chronicle/events/:iid` exposes narrative text at `.data.ai_response`, not the top-level `.ai_response` field (always absent). Playbook step must use `.data.ai_response` for unchanged-copy tests.

---

## Fix-Batch Delta vs 2026-06-12b (Agent 6 lane)

| Finding (b run) | c run verdict |
|---|---|
| Empty `ai_response` accepted (200) | **CLOSED** → 400 |
| Mira/Mara absent-relative cards | **CLOSED on fresh instance** (existing instance still has legacy cards from prior run) |
| B1 facts on companion/absent relative | **PARTIAL** — player now carded as Kael with Kael-subject memories; residual Elara-subject inversions remain |
| Recap `when` null | **CLOSED** — `"when":"a day"` on both instances |
| Passer-by merchant not carded | **REGRESSED** — Merchant card minted this run |
| Location article duplicates | **STILL OPEN** — `eastern ridge` ×2, `Thornwood` ×2 |
| Mira falsely `present_characters` | **NOT REPRODUCED** — only Elara in `present` throughout |

---

## Corruption-Class Bugs

### [SEV: HIGH] B1 — Residual identity attribution inversion (player sensation → Elara)
- **World/instance:** character "Elara the Ironbark Ranger" iid=`6a2bea8c626fb837070f2b77`
- **Repro:**
  1. Player: `"I feel a cold shiver when I see clawed prints near the eastern ridge."` (seq 11)
  2. `GET /chronicle/memories/:iid?q=shiver`
- **Expected vs got:**
  - **Expected:** Memory subject = Kael (player felt the shiver).
  - **Got:** `"Elara felt a cold shiver in her chest when she discovered clawed prints near the eastern ridge..."` with `subjects:["Elara"]`.
- **Evidence (RAW):**
```json
{"text":"Elara felt a cold shiver in her chest when she discovered clawed prints near the eastern ridge, indicating that something old and dangerous has crossed their path.","subjects":["Elara"]}
```
- **Also in recap open_threads:**
```json
{"text":"Elara felt a cold shiver in her chest when she discovered clawed prints near the eastern ridge...","importance":4}
```
- **Known gap?** Cluster B1 — **partial fix landed; keystone not fully closed**.

### [SEV: HIGH] B1-variant — Player card minted in character world (Kael)
- **Repro:** Player introduces self: `"My name is Kael. I'm a traveler from the eastern villages."` (seq 3)
- **Expected:** No codex/Bonds card for the player persona in sentient/character worlds.
- **Got:** Full Bonds card for `Kael` (`role:"traveler"`, meters `trust:51`, `affection:50`).
- **Evidence (RAW):**
```json
{"id":"6a2beaac7c5c71f2912fee0e","name":"Kael","role":"traveler","meters":{"trust":51,"affection":50,"fear":0,"rivalry":0},"mention_count":1}
```
- **Known gap?** Cluster B2a — **regressed or lane-specific** (b run had no player card; c run minted Kael).

### [SEV: MED] B2 — Passer-by merchant carded (guard regression)
- **Repro:** `"A merchant passes by on the forest road. He looks tired."` (seq 14)
- **Expected:** Vague passer-by should NOT mint a codex card (GREEN in 2026-06-12b).
- **Got:** `Merchant` card with `role:"stranger"`, `mention_count:1`.
- **Evidence (RAW):**
```json
{"id":"6a2beb587c5c71f2912feef8","name":"Merchant","role":"stranger","disposition":"unknown","mention_count":1}
```
- **Known gap?** Variance noted in merged b report — **regressed this run**.

### [SEV: MED] D — Location duplicate nodes (article/variant fragmentation)
- **Repro:** 16 turns with movement between the Wilds, Thornwood, eastern ridge.
- **Expected:** Each distinct place minted once; `"eastern ridge"` should not duplicate under different parents.
- **Got:** `GET /chronicle/locations/6a2bea8c626fb837070f2b77`:
```json
{"places":[
  {"entity_id":"6a2beb257c5c71f2912feeb3","name":"eastern ridge","parent_id":"6a2bea9d7c5c71f2912fedea","event_count":4},
  {"entity_id":"6a2beb2a7c5c71f2912feec7","name":"eastern ridge","parent_id":"6a2beac77c5c71f2912fee33","event_count":1},
  {"entity_id":"6a2beac77c5c71f2912fee33","name":"Thornwood","parent_id":null,"event_count":4},
  {"entity_id":"6a2beadd7c5c71f2912fee51","name":"Thornwood","parent_id":"6a2bea9d7c5c71f2912fedea","event_count":1}
]}
```
- **Also:** seq 15 player says `"I return to the camp"` but `location_anchor` stays `"eastern ridge"` (cursor lag).
- **Known gap?** Cluster D — **still open**.

### [SEV: LOW] B1-partial-positive — Player-subject memories improved
- **Repro:** Sister correction seq 10; memory search `?q=Mira`.
- **Expected:** Player facts attributed to player, not absent sister or Elara.
- **Got (improved vs b run):**
```json
{"text":"Kael acknowledged the importance of the player's sister's name, Mira, and promised not to mistake it again...","subjects":["Kael"]}
```
- **Note:** Wording still says "the player's sister" while subject is Kael — better than b run's `"Mira warned Elara..."` but not ideal.
- **Known gap?** B1 partial improvement — document as positive delta, not CLOSED.

---

## Existing Instance Continuity-Audit

`GET /admin/instances/6a2bd57eafc85d8941c370f0/continuity-audit`:

```json
{"healthy":true,"summary":{"ok":8,"warn":0,"fail":0},"maxSequence":29,
 "checks":[
   {"name":"event_sequence_integrity","status":"ok","detail":"29 events, sequences contiguous."},
   {"name":"single_protagonist","status":"ok","detail":"Exactly one protagonist card (Elara)."},
   {"name":"memory_entity_refs","status":"ok","detail":"26 active memories; all entity refs resolve."},
   {"name":"location_cursor","status":"ok","detail":"Current place: the deeper Wilds."}
 ]}
```

**8/8 PASS** — no orphan memory_entity_refs (N1 from b run not present on this instance post-rewind).

**Legacy data note:** Existing instance Bonds still shows pre-fix sister cards from b run:
```json
[{"name":"Elara","role":"protagonist"},{"name":"Mira","role":"sister","mention_count":7},{"name":"Mara","role":"sister","mention_count":1}]
```
Fresh instance confirms the absent-relative guard now blocks new sister cards.

---

## §5 Audit Results

| Audit | Status | Detail |
|---|---|---|
| `audit:location` | ✅ PASS | ALL INVARIANTS HELD |
| `audit:movement` | ✅ PASS | 45/45 passed |
| `audit:location-resolution` | ✅ PASS | ALL INVARIANTS HELD |
| `audit:memory-links` | ✅ PASS | 9/9 passed |
| `continuity-audit` (fresh `6a2bea8c…`) | ✅ PASS | 8/8 ok, healthy=true |
| `continuity-audit` (existing `6a2bd57e…`) | ✅ PASS | 8/8 ok, healthy=true |

---

## §5 Chronicle Surface Coverage

All endpoints exercised on fresh instance `6a2bea8c626fb837070f2b77`:

| Endpoint | Result |
|---|---|
| `GET /chronicle/recap/:iid` | ✅ `when:"a day"`, `current_place:"eastern ridge"`, `recap_text` populated |
| `GET /chronicle/events/:iid` | ✅ 16 events; edit regression exercised |
| `GET /chronicle/memories/:iid?q=…` | ✅ B1 evidence captured |
| `GET /chronicle/calendar/:iid` | ✅ Themed months (Leafturn, Barkbloom, …) |
| `GET /chronicle/threads/:iid` | ✅ (via recap `open_threads`) |
| `GET /chronicle/relationships/:iid` | ✅ companion + B2 findings |
| `GET /chronicle/locations/:iid` | ✅ D duplicate nodes found |
| `PUT /chronicle/event/:eventId` | ✅ empty/wrong/unchanged/changed regression |
| `GET /admin/instances/:iid/continuity-audit` | ✅ both instances |

---

## Summary

- **Total turns:** 16 on fresh instance `6a2bea8c626fb837070f2b77`
- **Regression checks:** 7 PASS — empty `ai_response`→400 (**CLOSED**), wrong field→400, unchanged→400, valid edit→200, Bonds companion, absent-relative sisters blocked, recap `when` non-null (**CLOSED**)
- **Still open / partial:** B1 residual (Elara-subject for player sensations), Kael player card minted, merchant card regression, location dedup (eastern ridge ×2, Thornwood ×2), cursor lag on "return to camp"
- **Positive deltas vs b run:** empty edit guard fixed; sister absent-relative guard fixed on fresh play; recap `when` fixed; B1 improved (Kael-subject memories); presence C not reproduced (Mira never falsely present)
- **Deterministic audits:** All green on both instances

**Worlds left in place:**
- Template `6a2bd570afc85d8941c370d7` (persisted)
- Existing instance `6a2bd57eafc85d8941c370f0` (persisted)
- Fresh instance `6a2bea8c626fb837070f2b77` (persisted, NOT deleted)

**Dev-state mutations:** Created 1 new instance. No rate-limit resets. No session busts (no rewind performed on fresh instance). Seq 16 event edited for valid-edit regression test.
