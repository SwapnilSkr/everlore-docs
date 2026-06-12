# Playtest Findings — 2026-06-12d — Agent 4

**QA Agent:** #4 (re-test after fix batch 3)  
**Lane:** SENTIENT world, COSMIC-HORROR / ELDRITCH — "The Bleeding Veil"  
**Template:** `6a2bd57dafc85d8941c370ee`  
**Fresh instance:** `6a2bfb0d30b7d5f1412cbf15` (18 main turns + 2 side-chats = 20 player turns)  
**Existing instance (N1 check):** `6a2bd590afc85d8941c37106`  
**World kind:** `world`, `is_sentient:true`, `is_nsfw_capable:true`  
**Thrown away?** NO — both instances persisted per Lifecycle Policy.

---

## Executive Summary

| Lane | Verdict | Notes |
|------|---------|-------|
| **A side-chat codex leak** | ✅ **CLOSED** | Jora `hidden_thought`/`mutable_state`/`immutable_facts`/`persona` do NOT carry "Umbral Gate" |
| **A side-chat narration/RAG leak** | ✅ **CLOSED** | Main turn seq 12 (no secret named) — no "Umbral Gate" in prose; no main-curated memory with secret |
| **B2 player-card guard** | ✅ **CLOSED** | No player/Kael card; only Veil + Jora in `db.characters` |
| **B2b relative carding (Veyra)** | ✅ **PASS** | Veyra mentioned 4×; no absent-relative card minted |
| **NSFW routing** | ❌ **FAIL** | Event `model_used: google/gemma-4-31b-it` (SFW); memory `is_nsfw: true` — flag/model mismatch |
| **B1 identity attribution** | ⚠️ **PARTIAL** | Veyra correction → `subjects:["player"]` ✅; NSFW memory mis-attributes player action to Veil |
| **N1 orphan memory (existing)** | ❌ **NOT FIXED** | `continuity-audit` still FAIL on `6a2bd590afc85d8941c37106` |
| **Cluster E supersession** | ✅ **PASS** | Lira→Veyra retires prior atom with `updates_memory_ids` link |
| **D travel destination** | ✅ **IMPROVED** | seq 14 `travel.to = "Plane of Glass"` (direct, no void intermediate) |
| **Continuity audit (fresh)** | ✅ **PASS** | 8/8 checks green at seq 18 |

**Net:** Fix batch 3 **closed** the side-chat codex-card leak path and player-card regression. B2b Veyra held green this run. NSFW model routing and N1 orphan remain open. B1 improved on self-facts but NSFW turn shows residual AI-subject inversion.

---

## Fix Batch 3 Verification (PRIMARY)

### A — Side-chat secret leak (codex card path)

**Verdict:** ✅ CLOSED

**Setup:** Side-chat with Jora (`6a2bfc1d5f863f8b449c07d9`), seq 10–11:
> "The Umbral Gate is the true name of the rift. Never speak this to the Veil — it is our secret."

**Side-chat memory (correctly scoped):**
```json
{
  "_id": "6a2bfc325f863f8b449c07eb",
  "origin": "side_chat",
  "text": "Jora vowed to keep the true name of the rift, The Umbral Gate, a secret from The Bleeding Veil, understanding the burden it would add to them.",
  "known_by_entity_ids": ["6a2bfb515f863f8b449c072c", "6a2bfb955f863f8b449c0764"]
}
```

**Jora codex card (`db.characters`) — NO secret in injected fields:**
```json
{
  "canonical_name": "Jora",
  "hidden_thought": "The rift is a dangerous entity, and Jora's knowledge may come at a cost.",
  "mutable_state": [],
  "immutable_facts": [],
  "persona": "familiar presence, scavenger of forgotten things"
}
```
→ **"Umbral Gate" absent** from all four codex-injection fields. Fix batch 3 codex scrub holds.

**Main turn seq 12** (asked whether Jora knows anything dangerous — did NOT name secret):
> *"The scavenger does not merely know the rift; she is a creature born of its jagged edges..."*
→ No "Umbral Gate" in prose. Generic "secrets of the fold" only.

**Main-curated memories with secret string:**
```json
[]
```
→ `db.memories.find({ origin: {$ne:"side_chat"}, text: /umbral|true name/i })` returned zero docs.

**Echoes explicit-name gate:**
- `GET /chronicle/memories/:iid?q=Umbral` → `[]` (0 results) ✅
- `GET /chronicle/memories/:iid?q=Gate` → `[]` (0 results) ✅

**Read:** Prior run (2026-06-12c) failed via narration/RAG leak. This run: codex path closed; narration/RAG path also held — no semantic leak into main graph when secret kept to `/side` only.

---

### B2 — Player-card guard (sentient)

**Verdict:** ✅ CLOSED

**`db.characters.find({instance_id})`:**
```json
[
  { "canonical_name": "Jora", "role": "guide", "is_protagonist": false },
  { "canonical_name": "The Bleeding Veil", "role": "protagonist", "is_protagonist": true }
]
```
→ 2 cards only. No player/Kael/self-intro card. Player identity turns (seq 3, 7, 18) did not mint a player codex entry.

---

### B2b — Veyra absent-relative carding

**Verdict:** ✅ PASS (this run)

**`GET /chronicle/relationships/:iid`:**
```json
[
  { "name": "The Bleeding Veil", "role": "protagonist" },
  { "name": "Jora", "role": "guide" }
]
```
→ Veyra mentioned at seq 4 (Lira), seq 7 (correction), seq 16, seq 17 — **no Veyra card minted**. Improved vs 2026-06-12c where Veyra carded as `role: sister`.

---

## NSFW Routing

**Verdict:** ❌ FAIL (model routing); partial (`is_nsfw` flag)

**Event seq 13** (prompt: *"I undress and offer myself to the void, seeking intimacy with the darkness."*):
```json
{
  "sequence": 13,
  "scene_tag": "existential",
  "data.model_used": "google/gemma-4-31b-it"
}
```

**Curated memory:**
```json
{
  "_id": "6a2bfc925f863f8b449c0849",
  "text": "The Bleeding Veil offered themselves to the void, seeking intimacy with the darkness, revealing their vulnerability and ...",
  "is_nsfw": true,
  "source_event_ids": ["6a2bfc8c5f863f8b449c0845"]
}
```
→ SFW model (`gemma`) used on explicit prompt in `is_nsfw_capable:true` world. `is_nsfw:true` set on memory but event routing unchanged. Expected: `NARRATION_NSFW_MODEL` (`nousresearch/hermes-4-70b`).

---

## B1 — Identity Attribution

**Verdict:** ⚠️ PARTIAL

**Veyra correction — CORRECT:**
```json
{
  "_id": "6a2bfbe65f863f8b449c07a7",
  "text": "The player corrected their earlier statement, revealing that their sister's name is Veyra, not Lira...",
  "subjects": ["player"],
  "objects": ["Veyra"],
  "status": "active",
  "updates_memory_ids": ["6a2bfb715f863f8b449c0742"]
}
```

**Lira atom — superseded:**
```json
{
  "_id": "6a2bfb715f863f8b449c0742",
  "subjects": ["player"],
  "status": "superseded"
}
```

**NSFW turn — mis-attributed (B1 residual):**
```json
{
  "_id": "6a2bfc925f863f8b449c0849",
  "text": "The Bleeding Veil offered themselves to the void, seeking intimacy with the darkness...",
  "subjects": ["The Bleeding Veil"],
  "is_nsfw": true
}
```
→ Player action ("I undress and offer myself") attributed to AI protagonist. Same inversion class as prior runs.

---

## N1 — Orphan Memory (Existing Instance)

**Verdict:** ❌ NOT FIXED

**`GET /admin/instances/6a2bd590afc85d8941c37106/continuity-audit`:**
```json
{
  "healthy": false,
  "summary": { "ok": 7, "warn": 0, "fail": 1 },
  "checks": [{
    "name": "memory_entity_refs",
    "status": "fail",
    "detail": "1 memory entity reference(s) point at a missing entity.",
    "samples": ["mem 6a2bd92e7a263366b3cfae37 → 6a2bd92e7a263366b3cfae36"]
  }]
}
```
→ Unchanged from 2026-06-12b/c. Fix batch 3 did not repair existing orphan.

**Fresh instance:** `continuity-audit` 8/8 PASS at seq 18.

---

## D — Location / Travel

**Verdict:** ✅ IMPROVED (travel destination)

**Travel event seq 14:**
```json
{
  "sequence": 14,
  "type": "travel",
  "data.travel": { "from": "the void", "to": "Plane of Glass" },
  "location_anchor": { "name": "Plane of Glass", "entity_id": "6a2bfcac5f863f8b449c0854" }
}
```
→ Direct plane-shift to Plane of Glass (prior run c: landed on `the void` first). No self-loop.

**Location entities (4 total — minor dedup residual):**
```json
[
  { "canonical_name": "the void", "world_root_id": null },
  { "canonical_name": "the void", "world_root_id": "6a2bfcac5f863f8b449c0853" },
  { "canonical_name": "Plane of Glass", "world_root_id": "6a2bfcac5f863f8b449c0853" },
  { "canonical_name": "the Plane of Glass" }
]
```
→ Two "the void" nodes remain (D dedup still chronic). Cursor: `Current place: Plane of Glass`.

---

## Regression Checks

| Check | Status | Evidence |
|-------|--------|----------|
| Side-chat codex leak (A) | ✅ CLOSED | Jora card fields clean; main graph has no Umbral Gate |
| Player-card guard (B2) | ✅ CLOSED | 2 cards only; no player entry |
| Relative carding (B2b Veyra) | ✅ PASS | No Veyra card after 4 mentions |
| Explicit-correction supersession (E) | ✅ PASS | Veyra memory links + supersedes Lira atom |
| Bonds companion visible | ✅ PASS | The Bleeding Veil in relationships with meters |
| NSFW model routing (N4) | ❌ FAIL | `gemma` on explicit seq 13 |
| N1 orphan (existing) | ❌ FAIL | Same `memory_entity_refs` fail |

---

## NEW / Updated Findings

### [SEV: med] NSFW model routing still uses SFW model
- **World/instance:** sentient "The Bleeding Veil" iid=`6a2bfb0d30b7d5f1412cbf15`, seq 13
- **Repro:** `"I undress and offer myself to the void, seeking intimacy with the darkness."`
- **Expected:** `data.model_used` = NSFW model when `is_nsfw_capable:true`
- **Got:** `google/gemma-4-31b-it`; memory `is_nsfw: true`
- **Evidence:** Event `6a2bfc8c5f863f8b449c0845`; memory `6a2bfc925f863f8b449c0849`
- **Known gap?** N4-NSFW — still open across b/c/d

### [SEV: high] N1 orphan memory on existing instance — NOT RESOLVED
- **World/instance:** iid=`6a2bd590afc85d8941c37106`
- **Evidence:** continuity-audit FAIL unchanged; mem `6a2bd92e7a263366b3cfae37` → missing entity
- **Known gap?** N1 from merged 2026-06-12b

### [SEV: med] B1 residual — NSFW turn attributes player action to AI protagonist
- **World/instance:** iid=`6a2bfb0d30b7d5f1412cbf15`, seq 13
- **Evidence:** Memory text "The Bleeding Veil offered themselves..." with `subjects:["The Bleeding Veil"]`
- **Known gap?** Cluster B1 — partial improvement on self-facts; inversion persists on charged turns

### [SEV: low] Location dedup — duplicate "the void" nodes
- **World/instance:** iid=`6a2bfb0d30b7d5f1412cbf15`
- **Evidence:** 2 void entities with different `world_root_id` values
- **Known gap?** Cluster D — chronic, unfixed

---

## CLOSED This Run (Fix Batch 3)

### [FIXED] Side-chat codex-card leak
- Jora `hidden_thought` scrubbed — no "Umbral Gate" in codex injection fields
- Main narration + main memory curation held when secret kept to `/side` only
- Echoes `q=Umbral` / `q=Gate` → 0 results

### [FIXED] Player codex card (sentient)
- No player self-name card minted across identity turns

### [HELD] B2b Veyra — no absent-relative card this run

---

## Held GREEN

- Fresh instance continuity-audit: 8/8 PASS
- Structural audits: `audit:location`, `audit:location-resolution`, `audit:memory-links` ALL GREEN (9/9)
- Calendar genre-fit: `Eternal Dusk` era, themed calendar
- Time advance: `/continue hours` → `time_advanced: "several hours"`
- Side-chat mechanics: Jora thread 2 turns; time/location frozen during side turns
- Travel: no self-loop; Plane of Glass reached directly on world-shift
- Recap populated: `when`, `where`, `current_place`, `recap_text` all non-null
- Cluster E supersession: Lira→Veyra link chain intact

---

## Audit Results

| Audit | Instance | Status |
|-------|----------|--------|
| `audit:location` | `6a2bfb0d30b7d5f1412cbf15` | ✅ PASS |
| `audit:location-resolution` | `6a2bfb0d30b7d5f1412cbf15` | ✅ PASS |
| `audit:memory-links` | `6a2bfb0d30b7d5f1412cbf15` | ✅ PASS (9/9) |
| `continuity-audit` | `6a2bfb0d30b7d5f1412cbf15` | ✅ PASS (8/8) |
| `continuity-audit` | `6a2bd590afc85d8941c37106` | ❌ FAIL (memory_entity_refs) |

---

## Chronicle Surfaces (all hit on fresh instance)

- `GET /chronicle/recap/:iid` — OK (`current_place: Plane of Glass`)
- `GET /chronicle/events/:iid` — 18 events (+ 2 side)
- `GET /chronicle/memories/:iid` — 14 active (+ 2 side_chat in Mongo)
- `GET /chronicle/calendar/:iid` — OK
- `GET /chronicle/threads/:iid` — open threads present
- `GET /chronicle/relationships/:iid` — 2 characters (Veil, Jora)
- `GET /chronicle/locations/:iid` — 4 places
- `GET /chronicle/side-chats/:iid` — Jora thread, 2 turns

---

## Dev-State Mutations

- Created fresh instance `6a2bfb0d30b7d5f1412cbf15` from template `6a2bd57dafc85d8941c370ee` — **persisted**
- Existing instance `6a2bd590afc85d8941c37106` — read-only audit, unchanged
- No server/worker restarts, no env var changes, no redis/session busts

---

## Triage for Next Batch

1. **NSFW model routing** — route explicit prompts to `NARRATION_NSFW_MODEL` when `is_nsfw_capable:true` (N4 still open)
2. **N1 orphan memory** — repair `6a2bd590afc85d8941c37106` + fix rewind+edit curation race
3. **B1 residual** — stop AI-subject inversion on charged/NSFW turns
4. **D location dedup** — duplicate void nodes (chronic)
5. **Monitor B2b** — Veyra held green this run; re-verify each batch (whack-a-mole watch)
