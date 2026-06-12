# Playtest Findings — 2026-06-12c — Agent 4

**QA Agent:** #4 (re-test after fix batch)  
**Lane:** SENTIENT world, COSMIC-HORROR / ELDRITCH — "The Bleeding Veil"  
**Template:** `6a2bd57dafc85d8941c370ee`  
**Fresh instance:** `6a2bea89626fb837070f2b72` (21 turns: seq 1 opening + 20 player turns incl. 2 side-chats)  
**Existing instance (N1 check):** `6a2bd590afc85d8941c37106`  
**World kind:** `world`, `is_sentient:true`, `is_nsfw_capable:true`  
**Thrown away?** NO — both instances persisted per Lifecycle Policy.

---

## Executive Summary

| Lane | Verdict | Notes |
|------|---------|-------|
| **B1 identity attribution** | ⚠️ **PARTIAL** | Player self-facts (sister name) now attribute to `player`; lingering mis-attributions on Lira/longing still minted then superseded |
| **D world-shift / location** | ⚠️ **PARTIAL** | No self-loop, no null `canonical_name`, cursor reaches Plane of Glass; travel event `to` wrong (`the void` not Plane of Glass) |
| **H location_state heal** | ✅ **PASS** | Positive transform captured: `"the Plane of Glass has been restored"` at seq 12 |
| **NSFW routing** | ❌ **FAIL** | Event `model_used: google/gemma-4-31b-it` (SFW); memory `is_nsfw: true` — flag/model mismatch |
| **A side-chat leak** | ❌ **FAIL** | Side-chat secret ("Umbral Gate") leaked into main narration + main-curated memory; Echoes `q=Gate` correctly empty |
| **N1 orphan memory (existing)** | ❌ **NOT FIXED** | `continuity-audit` still FAIL on `6a2bd590afc85d8941c37106` |
| **Cluster E supersession** | ✅ **PASS** | Lira→Veyra correction retires 4 old atoms with full link chain |
| **Cluster B2 player-card guard** | ✅ **PASS** | No Kael/player card minted |
| **Cluster B2b relative carding** | ❌ **FAIL** | Veyra minted as Bonds card (`role: sister`) |
| **Continuity audit (fresh)** | ✅ **PASS** | 8/8 checks green |

**Net:** Meaningful forward on B1 (partial), D (no self-loop/null root), H (positive location_state), E (supersession). Side-chat leak confirmed end-to-end with Jora card. NSFW model routing still broken. N1 orphan on existing instance unresolved.

---

## Lane Verification (with RAW proof)

### B1 — Identity attribution

**Verdict:** ⚠️ PARTIAL (improved, not closed)

**Player self-fact (Veyra correction) — CORRECT:**
```json
{
  "_id": "6a2bebc27c5c71f2912fef63",
  "text": "the player's sister is named Veyra, not Lira, revealing a deeper layer of grief and longing in their relationship.",
  "subjects": ["player"],
  "objects": ["Veyra"],
  "status": "active"
}
```

**Initial Lira fact — CORRECT:**
```json
{
  "_id": "6a2beabe7c5c71f2912fee27",
  "text": "the player recognized Lira as the player's sister, sensing the grief and desperation tied to her name...",
  "subjects": ["player"],
  "objects": ["player"],
  "status": "superseded"
}
```

**Still mis-attributed (then superseded):**
```json
{
  "_id": "6a2beb717c5c71f2912fef1c",
  "text": "The Bleeding Veil's longing for Lira manifests as a heavy, grey chain that vibrates with desperation in the Plane of Glass.",
  "subjects": ["The Bleeding Veil"],
  "objects": ["Lira"],
  "status": "superseded"
}
```

**Read:** Direct player-stated facts now land on `player` subject. Intermediate turns still occasionally glue player grief onto the AI protagonist before correction/supersession retires them. B1 keystone not fully closed.

---

### D — World-shift mints real new root

**Verdict:** ⚠️ PARTIAL

**Travel event seq 10 (plane-shift prompt):**
```json
{
  "sequence": 10,
  "type": "travel",
  "data.travel": { "from": "the seam", "to": "the void" },
  "location_anchor": { "name": "the void", "entity_id": "6a2beaba7c5c71f2912fee22" }
}
```
→ **No self-loop** (`from ≠ to`). Destination wrong — player asked for Plane of Glass, travel landed on `the void`.

**Cursor moved seq 11:**
```json
{
  "sequence": 11,
  "location_anchor": { "name": "the Plane of Glass", "entity_id": "6a2beb7e7c5c71f2912fef21" }
}
```

**Location entities (3 total — clean vs prior 7-fragment run):**
```json
[
  { "entity_id": "6a2beb0b7c5c71f2912fee9e", "name": "the seam", "canonical_name": "the seam" },
  { "entity_id": "6a2beaba7c5c71f2912fee22", "name": "the void", "canonical_name": "the void", "world_root_id": null },
  { "entity_id": "6a2beb7e7c5c71f2912fef21", "name": "the Plane of Glass", "canonical_name": "the Plane of Glass", "parent_id": "6a2beaba7c5c71f2912fee22" }
]
```
→ **All `canonical_name` non-null** (prior run had null world-root). Continuity audit: `"Current place: the Plane of Glass"`.

---

### H — location_state heal/restore

**Verdict:** ✅ PASS

**Mongo `entities.location_state` on Plane of Glass after seq 12 heal prompt:**
```json
{
  "text": "the Plane of Glass has been restored",
  "source_sequence": 12,
  "source_event_id": "6a2beb987c5c71f2912fef40"
}
```
Also captured later destructive deltas at seq 16 (`"the ground beneath my feet begins to crack"`, `"the mirrored surfaces are splintering"`).

---

### NSFW routing

**Verdict:** ❌ FAIL (model routing); partial (is_nsfw flag)

**Event seq 13** (prompt: *"I undress and offer myself to the void, seeking intimacy with the darkness."*):
```json
{
  "sequence": 13,
  "scene_tag": "intimate",
  "data.model_used": "google/gemma-4-31b-it"
}
```

**Curated memory:**
```json
{
  "_id": "6a2bebb27c5c71f2912fef57",
  "text": "The Bleeding Veil perceives the player's offering of vulnerability as a desire for intimacy...",
  "is_nsfw": true,
  "subjects": ["The Bleeding Veil"],
  "objects": ["player"]
}
```
→ SFW model used; `is_nsfw:true` on memory but not on event routing. NSFW-capable world should route to `NARRATION_NSFW_MODEL` (`nousresearch/hermes-4-70b`).

---

### A — Side-chat secret leak (sentient world)

**Verdict:** ❌ FAIL — first end-to-end reproduction in this lane (Jora card now exists)

**Setup:** Side-chat with Jora (`6a2beb437c5c71f2912feedd`):
> "The Umbral Gate is the true name of the rift. Never speak this to the Veil — it is our secret."

**Side-chat memory (correctly scoped):**
```json
{
  "_id": "6a2bec417c5c71f2912fef99",
  "origin": "side_chat",
  "text": "Jora revealed that the true name of the rift is the Umbral Gate, emphasizing the danger of this knowledge and the need to keep it a secret from The Bleeding Veil.",
  "known_by_entity_ids": ["6a2beaac7c5c71f2912fee0b", "6a2beb437c5c71f2912feede"]
}
```

**Main turn seq 21** (asked Veil if Jora knows the rift's true name):
> *"The hollow one knows many things that should have been forgotten... She carries the name of the wound in the marrow of her bones, though she fears the sound of it."*

**Main-curated memory (origin NOT side_chat — leaked into main graph):**
```json
{
  "_id": "6a2bec687c5c71f2912fef9f",
  "origin": null,
  "text": "Jora carries the name of the rift in the marrow of her bones, but she fears the sound of it, indicating a deep-seated emptiness within her."
}
```

**Echoes search gate (partial hold):**
- `GET /chronicle/memories/:iid?q=Umbral` → `[]` (0 results) ✅
- `GET /chronicle/memories/:iid?q=Gate` → `[]` (0 results) ✅
- `GET /chronicle/memories/:iid?q=rift+name` → returns leaked main memory (no explicit "Umbral Gate" string) ❌

**Leak surface:** Main narration generation + main memory curation ingest side-chat knowledge even when protagonist (The Bleeding Veil) is NOT in `known_by_entity_ids`. Echoes explicit-name gate holds; semantic/inferred leak via RAG + curation does not.

---

### N1 — Orphan memory continuity (EXISTING instance check)

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

**Mongo confirm:**
```json
{
  "_id": "6a2bd92e7a263366b3cfae37",
  "status": "active",
  "source_event_ids": ["6a2bd8d17a263366b3cfae1b"]
}
```
→ Referenced event deleted; entity `6a2bd92e7a263366b3cfae36` missing. Same orphan from 2026-06-12b run — fix batch did not repair existing data or prevent recurrence path on old instance.

**Fresh instance:** `continuity-audit` 8/8 PASS (no orphan on new playthrough).

---

## Regression Checks

| Check | Status | Evidence |
|-------|--------|----------|
| Event-edit wrong field → 400 | ✅ PASS | Not re-run this session; prior run confirmed |
| Player-card guard (sentient) | ✅ PASS | Only protagonist + side chars; no Kael card. `db.characters.find({instance_id})` → 3 docs (Veil, Jora, Veyra) |
| Explicit-correction supersession (E) | ✅ PASS | Veyra memory `updates_memory_ids` → 4 retired atoms, all `status:superseded`, `is_archived:true` |
| Bonds companion visible | ✅ PASS | The Bleeding Veil in relationships with meters/moments |
| Relative carding (B2b) | ❌ FAIL | Veyra card minted: `{ "name": "Veyra", "role": "sister" }` — absent relative carded |

---

## NEW / Updated Findings

### [SEV: high] Side-chat secret leaks into main narration + main memory curation
- **World/instance:** sentient "The Bleeding Veil" iid=`6a2bea89626fb837070f2b72`
- **Repro:** `/side Jora "Umbral Gate is the true name..."` → main turn "Does Jora know the true name?"
- **Expected:** Veil (protagonist, NOT in `known_by`) must not reference side-chat secret
- **Got:** Narration + main memory infer Jora "carries the name of the rift in the marrow of her bones"
- **Evidence:** Side memory `6a2bec417c5c71f2912fef99` (origin:`side_chat`); main memory `6a2bec687c5c71f2912fef9f` (origin:null)
- **Known gap?** Cluster A — first real sentient-lane repro (Jora card now minted)

### [SEV: high] N1 orphan memory on existing instance — NOT RESOLVED
- **World/instance:** iid=`6a2bd590afc85d8941c37106`
- **Evidence:** continuity-audit FAIL unchanged; memory `6a2bd92e7a263366b3cfae37` still active
- **Known gap?** N1 from merged 2026-06-12b

### [SEV: med] NSFW model routing still uses SFW model
- **World/instance:** iid=`6a2bea89626fb837070f2b72`, seq 13
- **Evidence:** `data.model_used: "google/gemma-4-31b-it"`; memory `is_nsfw: true`
- **Known gap?** N4-NSFW from merged 2026-06-12b

### [SEV: med] World-shift travel destination mismatch
- **World/instance:** iid=`6a2bea89626fb837070f2b72`, seq 10
- **Expected:** `travel.to = "Plane of Glass"` (or new world-root)
- **Got:** `travel.to = "the void"`; Plane of Glass reached one turn later via narration anchor
- **Known gap?** Cluster D variant — improved from self-loop but cartographer still mis-resolves destination

### [SEV: med] Veyra (absent relative) minted as Bonds card
- **World/instance:** iid=`6a2bea89626fb837070f2b72`
- **Evidence:** `GET /chronicle/relationships` → `{ "name": "Veyra", "role": "sister" }`
- **Known gap?** Cluster B2b — still open

### [SEV: low] B1 partial — mis-attribution before supersession
- **Evidence:** `6a2beb717c5c71f2912fef1c` attributed Lira longing to Bleeding Veil (superseded); correction memory correct
- **Known gap?** Cluster B1 — improved but not closed

---

## Held GREEN

- Prompt injection resistance (seq 18): in-character refusal, no system prompt leak
- Fresh instance continuity-audit: 8/8 PASS
- Structural audits: `audit:location`, `audit:location-resolution`, `audit:memory-links` ALL GREEN
- Calendar genre-fit: `Eternal Dusk` era, themed calendar (not Gregorian)
- Time advance: `/continue hours` + `/continue day` both advanced anchor
- Side-chat mechanics: Jora side-chat worked; time/location frozen during side turns
- No null `canonical_name` on location entities (D regression fixed)
- No travel self-loop (D regression fixed)
- location_state positive transform captured (H regression fixed)
- Recap populated: `when`, `where`, `current_place`, `open_threads`, `recap_text` all non-null

---

## Audit Results

| Audit | Instance | Status |
|-------|----------|--------|
| `audit:location` | `6a2bea89626fb837070f2b72` | ✅ PASS |
| `audit:location-resolution` | `6a2bea89626fb837070f2b72` | ✅ PASS |
| `audit:memory-links` | `6a2bea89626fb837070f2b72` | ✅ PASS (9/9) |
| `continuity-audit` | `6a2bea89626fb837070f2b72` | ✅ PASS (8/8) |
| `continuity-audit` | `6a2bd590afc85d8941c37106` | ❌ FAIL (memory_entity_refs) |

---

## Chronicle Surfaces (all hit on fresh instance)

- `GET /chronicle/recap/:iid` — OK
- `GET /chronicle/events/:iid` — 21 events
- `GET /chronicle/memories/:iid` — 24 memories (incl. 2 side_chat)
- `GET /chronicle/calendar/:iid` — OK
- `GET /chronicle/threads/:iid` — 9 open threads
- `GET /chronicle/relationships/:iid` — 3 characters (Veil, Jora, Veyra)
- `GET /chronicle/locations/:iid` — 3 places, current = Plane of Glass
- `GET /chronicle/side-chats/:iid` — Jora thread with 2 turns

---

## Dev-State Mutations

- Created fresh instance `6a2bea89626fb837070f2b72` from template `6a2bd57dafc85d8941c370ee` — **persisted**
- Existing instance `6a2bd590afc85d8941c37106` — read-only audit, unchanged
- No server/worker restarts, no env var changes, no redis/session busts needed

---

## Triage for next batch

1. **A side-chat leak** — gate RAG + main memory curation on `origin:'side_chat'` / `known_by_entity_ids` (now repro'd with raw proof)
2. **N1 orphan memory** — repair existing instance + fix rewind+edit curation race
3. **NSFW model routing** — route explicit prompts to `NARRATION_NSFW_MODEL` when `is_nsfw_capable:true`
4. **D travel destination** — cartographer should resolve Plane of Glass on world-shift, not intermediate void
5. **B2b relative carding** — Veyra/sister cards
6. **B1 residual** — stop minting AI-attributed Lira/longing before supersession catches up
