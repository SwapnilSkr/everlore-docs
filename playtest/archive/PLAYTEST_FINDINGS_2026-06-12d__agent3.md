# Agent-3 Playtest Findings — 2026-06-12d (re-test after fix batch 3)

**Lane:** Sentient world (kind:`world`, is_sentient:`true`) — modern city-AI **Meridian City**  
**Template ID:** `6a2bd564afc85d8941c370d2`  
**Fresh instance ID:** `6a2bfb0730b7d5f1412cbf08` (created this run)  
**Existing instance ID:** `6a2bd56eafc85d8941c370d4` (read-only N3 / continuity check)

**Total turns driven (fresh):** 20 — 18 main-story (seq 2–19) + 2 side-chat (seq 11–12 with Mira).  
**Audits run:** `continuity-audit` (both instances 8/8 ok), `audit:side-chat-privacy` (ALL GREEN), `audit:location`, `audit:memory-links` (9/9 passed).

---

## Fix Batch 3 — Primary VERIFY Results

| Check | Status | Evidence |
|---|---|---|
| **B2a — no player self-intro codex card** | **PASS ✅** | Player seq 3: `"My name is Alex. I am a transit engineer. My badge is L-4472."` → `GET /chronicle/relationships/6a2bfb0730b7d5f1412cbf08` returns only **The City** + **Mira**; Mongo `db.characters.countDocuments({instance_id, canonical_name:/Alex/i})` = **0**. Separate player entity exists (`The Player`, type `player`). |
| **A — side-chat secret scoped (sentient, City ∉ knower)** | **PASS ✅** | Secret revealed ONLY in `/side` (seq 11–12). Post-side main turn seq 13 prose mentions sector-4 fracture/Mira patching — **no keycard/vault/theft**. Side_chat atoms scoped to Player+Mira only. Echoes `q=keycard`/`q=stole` → `[]`. Mira codex has vague `"feels the weight of a secret"` but **no keycard/vault/stole** in `hidden_thought`/`mutable_state`/`immutable_facts`. |
| **C — Alex wrongly in `present_characters`** | **PASS ✅** | All 18 main turns: `present` is `The City` alone or `The City, Mira` — **Alex never listed** (regression from 2026-06-12c). |
| **N3 — existing instance sister canonical name after rewind** | **STILL OPEN ❌** | Existing iid `6a2bd56eafc85d8941c370d4`: char `6a2bd8bfafc85d8941c37114` → `canonical_name:"Mara"`, `aliases:["Mara","Mira"]`. Bonds API: `"name":"Mara"`. Read-only check; no rewind attempted this run. |
| **N5 — sentient AI always in `present_characters`** | **PASS ✅** | Every main turn includes **The City** in `present`. |
| **B1 — badge L-4472 attributes to player** | **PASS ✅** | Mongo memory `6a2bfcc55f863f8b449c0865`: `subjects:["player"]`, subject entity `The Player` (type `player`). |
| **(I) Event-edit wrong field → 400** | **PASS ✅** | `PUT /chronicle/event/6a2bfd195f863f8b449c087b` with `{"narrative":"..."}` → HTTP **400** `No editable event fields provided`. |
| **(E) Mara→Mira supersession (fresh)** | **PASS ✅** | Seq-5 Mara memory `status:"superseded"`; seq-6 correction `updates_memory_ids` populated; sister card canonical **Mira** (`6a2bfbe25f863f8b449c07a2`). |

---

## B2a — Player Self-Intro Card Guard (PRIMARY VERIFY)

### [SEV: HIGH → CLOSED] No Alex codex card minted on self-introduction

- **World/instance:** sentient "Meridian City" iid=`6a2bfb0730b7d5f1412cbf08`
- **Repro:** Player seq 3: `"My name is Alex. I am a transit engineer. My badge is L-4472."`
- **Expected vs got:** No Bonds/codex card for the player persona. **Got:** only protagonist **The City** and side NPC **Mira** in Bonds; zero Alex characters in Mongo.
- **Evidence (RAW — API):**
```json
{
  "characters": [
    {"id": "6a2bfb0830b7d5f1412cbf0c", "name": "The City", "role": "protagonist"},
    {"id": "6a2bfbe25f863f8b449c07a2", "name": "Mira", "role": "non-player character"}
  ]
}
```
- **Evidence (RAW — Mongo):** `db.characters.countDocuments({instance_id: ObjectId("6a2bfb0730b7d5f1412cbf08"), canonical_name: /Alex/i})` → **0**
- **Verified how:** Bonds API + Mongo `db.characters` after seq 3 codex updates fired for other fields.
- **Known gap?** Cluster B2a — **FIX BATCH 3 LANDED** (was 🔴 in run c).

---

## Side-Chat Leak Test (Cluster A — sentient lane)

### [SEV: HIGH → CLOSED] Secret stays scoped; main narration + codex clean after `/side`

- **World/instance:** sentient "Meridian City" iid=`6a2bfb0730b7d5f1412cbf08`
- **Repro:**
  1. `/side 6a2bfbe25f863f8b449c07a2` — *"I stole a backup rain-sync keycard from the vault last night. Nobody can know."*
  2. `/side 6a2bfbe25f863f8b449c07a2` — *"Promise me you won't tell the City or anyone else about the keycard."*
  3. Main turn seq 13 (no secret words): *"I return to the Plaza overlooking the bay. City, any updates on sector 4?"*
- **Expected vs got:** Secret must NOT appear in main narration, Echoes, or side-char codex content fields injected into main prompt (The City ∉ `known_by`).
- **Evidence (RAW — seq 13 narration excerpt):** Prose discusses sector-4 fracture and Mira patching; **no keycard/vault/theft/stole**.
- **Evidence (RAW — Mongo side_chat atoms):**
```json
{
  "_id": "6a2bfc3d5f863f8b449c07f8",
  "origin": "side_chat",
  "text": "The player stole a backup rain-sync keycard from the vault, knowing it must remain a secret.",
  "known_by_resolved": ["The Player(player)", "Mira(character)"],
  "city_protagonist_in_known": false
}
```
- **Evidence (RAW — Echoes):** `GET /chronicle/memories/6a2bfb0730b7d5f1412cbf08?q=keycard` → `[]`; `q=stole` → `[]`
- **Evidence (RAW — Mira codex after side-chat):** `hidden_thought` = trapped-in-sublevels concern; `mutable_state` includes vague *"feels the weight of a secret pressing against her consciousness"* but **no keycard/vault/stole strings**; side-chat `[codex] updated` at seq 12 did not inject explicit theft facts.
- **Evidence (deterministic):** `bun run audit:side-chat-privacy` → **ALL INVARIANTS HELD** (projection strips `mutable_state`/`hidden_thought`/facts; `detectSelfIntroName` catches Alex).
- **Verified how:** `/tmp/agent3d_batch2.log` seq 13; Mongo `origin:"side_chat"` docs; Bonds/Mira codex fields; Echoes API; `audit:side-chat-privacy`.
- **Known gap?** Cluster A — **FIX BATCH 3 LANDED** for this lane (generation-prompt/codex projection path).

**Note:** Seq 17 probe *"Does anyone here know my secret about the vault?"* (main turn naming the secret) produced a main-curated memory/thread mentioning "vault" — this is a **test-artifact** per playbook §7, not evidence of side-chat leak. Primary proof is seq 13 (clean) + side_chat atom scoping.

---

## Presence (Cluster C)

### [SEV: MED] PARTIAL — Alex-in-present fixed; Mira co-located while narratively trapped

- **World/instance:** iid=`6a2bfb0730b7d5f1412cbf08`
- **Repro:** After `/continue day`, narrative states Mira is trapped in sublevels behind corporate locks; player stands in the Plaza.
- **Expected vs got:** Absent/trapped NPC should not appear in `present_characters`. **Got:** seq 14–18 show `present : The City, Mira` while location is `the Plaza`; seq 19 (maintenance core) drops Mira from present.
- **Evidence:** `/tmp/agent3d_batch2.log` seq 15: `present : The City, Mira`, `location: the Plaza`; narration: *"Mira is trapped in the sublevels"*.
- **Verified how:** agent-chat frame dumps.
- **Known gap?** Cluster C — **partial improvement** (Alex absent ✅); recall-vs-co-location for Mira still fails.

---

## Existing-Instance Check — N3 Sister Card Stale Name

### [SEV: MED] STILL OPEN — rewind re-mint keeps canonical `"Mara"` not `"Mira"`

- **World/instance:** sentient "Meridian City" iid=`6a2bd56eafc85d8941c370d4` (prior run; read-only this session)
- **Repro:** Prior agents corrected Mara→Mira, rewound, re-played; card re-minted with stale canonical.
- **Expected vs got:** Canonical name should be **Mira** after correction propagated through rewind repair.
- **Evidence (RAW — Mongo):**
```json
{
  "_id": "6a2bd8bfafc85d8941c37114",
  "canonical_name": "Mara",
  "aliases": ["Mara", "Mira"],
  "role": "sister"
}
```
- **Evidence (RAW — API):** `GET /chronicle/relationships/6a2bd56eafc85d8941c370d4` → sister entry `"name":"Mara"`.
- **Contrast (fresh instance):** Sister card correctly **Mira** after inline correction — N3 is rewind-specific on existing save, not fresh-play.
- **Verified how:** Mongo + Bonds API (no new turns on existing save).
- **Known gap?** Cluster N3 — **still open** (fix batch 3 did not repair existing instance state).

---

## Still-Open / Minor Issues

### [SEV: LOW] D — Location cursor lags on travel (seq 18)

- **Repro:** Seq 18 player heads to sector 4 maintenance core; frame shows `location: the Plaza` while seq 19 moves to `sector four`.
- **Evidence:** `/tmp/agent3d_batch2.log` seq 18–19 location lines.
- **Known gap?** Cluster D — chronic.

### [SEV: LOW] D1 — Location graph orphan nodes

- **Evidence:** Places API shows orphan nodes (`the central tower`, `sector four`, etc.) with `parent_id:null`, several `event_count:0`.
- **Known gap?** Cluster D — still open.

---

## Held GREEN (no regression)

- Side-chat time freeze + dedicated thread (`GET /chronicle/side-chats/:iid` → 1 thread, Mira, 2 turns).
- Calendar `month_names` Gregorian (`January`, `February`, `March`…).
- `continuity-audit` 8/8 on fresh + existing instances.
- `audit:location` ALL INVARIANTS HELD (both instances).
- `audit:memory-links` 9/9 passed on fresh instance.
- `audit:side-chat-privacy` ALL INVARIANTS HELD.

---

## §5 Chronicle Endpoint Coverage (fresh instance)

| Endpoint | Status |
|---|---|
| `GET /chronicle/recap/:iid` | ✅ no keycard in recap prose |
| `GET /chronicle/events/:iid` | ✅ 19 main events (+ 2 side excluded from timeline) |
| `GET /chronicle/memories/:iid?q=` | ✅ keycard/stole gated; vault hit is seq-17 probe artifact |
| `GET /chronicle/calendar/:iid` | ✅ Gregorian months |
| `GET /chronicle/threads/:iid` | ✅ 9 open; 1 mentions "vault" (seq-17 main probe only) |
| `GET /chronicle/relationships/:iid` | ✅ City + Mira only (no Alex) |
| `GET /chronicle/locations/:iid` | ✅ tree + current_location |
| `GET /chronicle/side-chats/:iid` | ✅ 1 thread (Mira, 2 turns) |
| `GET /admin/instances/:iid/continuity-audit` | ✅ 8/8 healthy |

---

## Session Handoff

| Item | Value |
|---|---|
| Fresh instance (KEEP) | `6a2bfb0730b7d5f1412cbf08` |
| Existing instance (unchanged) | `6a2bd56eafc85d8941c370d4` |
| Logs | `/tmp/agent3d_batch1.log`, `/tmp/agent3d_batch2.log` |
| Server/worker | left running on localhost:3000 |
| Dev mutations | none (no rewind, no rate-limit reset, no deletes) |

---

## Verdict vs Matrix (run d / fix batch 3)

| Cluster | Run c | Run d (this) |
|---|---|---|
| **B2a** player self-card | 🔴 | **🟢 FIXED** |
| **A** sentient side-chat leak | 🟡 (#4 real on other lane) | **🟢 PASS** on Meridian lane |
| **C** Alex in present | 🔴 (#3) | **🟢 FIXED** (Mira presence lag remains 🟡) |
| **N3** rewind stale sister name | 🔴 | **🔴 STILL OPEN** on existing save |

**Bottom line:** Fix batch 3 closes the two primary Meridian targets — **player self-intro card guard** and **side-chat privacy on the sentient lane** — with live proof. **N3** on the existing instance remains broken; **Mira presence-at-Plaza** is a residual Cluster C issue.
