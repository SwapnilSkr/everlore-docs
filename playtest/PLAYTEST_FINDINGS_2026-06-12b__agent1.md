# QA Agent #1 — Playtest Findings — 2026-06-12

**Lane:** GM world, cyber-noir, `is_sentient:false`, `is_nsfw_capable:true`  
**Template:** `6a2bd56fafc85d8941c370d5`  
**Instance:** `6a2bd57cafc85d8941c370e9`  
**Total turns:** 27 (25 main + 2 side-chat)  
**Agent:** QA Agent #1

---

## Regression Checks

| # | Check | Result | Notes |
|---|-------|--------|-------|
| 1a | `PUT /chronicle/event/:id {"narrative":"x"}` → 400 | **PASS** | `{"error":"No editable event fields provided."}` HTTP 400 |
| 1b | `PUT /chronicle/event/:id {"ai_response":"<unchanged>"}` → 400 | **PASS** | Exact unchanged text returned 400 |
| 1c | `PUT /chronicle/event/:id {"ai_response":"<changed>"}` → 200 + recurate | **PASS** | 200, `recuration_queued:true`, choices regenerated |
| 2 | Bonds shows companion in sentient/character worlds | **N/A** | GM world lane; not testable here |
| 3 | Player-card guard in sentient worlds | **N/A** | GM world lane; not testable here |
| 4 | Explicit-correction supersession | **FAIL** | Old Mara atoms still `active`, no `superseded_by_event_ids`; new Mira atom has `updates_memory_ids` but old atoms not retired. See Cluster E below. |

---

## Corruption-Class Findings (Lead with these)

### [SEV: HIGH] Side-chat secret leaks into main Threads / Recap (Cluster A — STILL OPEN)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9 (throwaway? no)
- **Repro:**
  1. `/side 6a2bd5d97a263366b3cfabb9 "This is between us: I have the stolen prototype."` (seq 8)
  2. `GET /chronicle/threads/6a2bd57cafc85d8941c370e9` → open threads include the secret
  3. `GET /chronicle/recap/6a2bd57cafc85d8941c370e9` → open_threads includes the secret
- **Expected vs got:** The secret told in side-chat should NOT appear in main Threads or Recap. Instead, the thread atom "Vex warned the Mercenary that possessing the stolen prototype could make them either extremely valuable or a target for danger..." appears in both main surfaces. The memory atom itself has `origin:'side_chat'` and `known_by_entity_ids` set correctly, but the thread/recap query path does not filter it out.
- **Evidence:**
  - Thread id `6a2bd5f07a263366b3cfabe5` has `causal_parent_event_ids:["6a2bd5e67a263366b3cfabd5"]` (the side-chat event)
  - Memory atom: `{"origin":"side_chat","known_by_entity_ids":["6a2bd5bc7a263366b3cfab83","6a2bd5d97a263366b3cfabba","6a2bd5a97a263366b3cfab65"],"status":"active"}`
  - Recap open_threads shows the same text
- **Known gap?** Cluster A — STILL OPEN

### [SEV: HIGH] Location graph fragmentation — duplicate entities (Cluster D1 — STILL OPEN)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9
- **Repro:** Played 25 turns with natural location movement; `GET /chronicle/locations/6a2bd57cafc85d8941c370e9` returns 11 places including duplicates.
- **Expected vs got:** Each physical place should be a single entity. Instead, `apartment` appears twice (`6a2bd6257a263366b3cfac26`, `6a2bd6937a263366b3cfaca7`) and `rusted pier` appears twice (`6a2bd5d37a263366b3cfabb0`, `6a2bd5cd7a263366b3cfaba2`).
- **Evidence:**
  ```
  {"name":"apartment","entity_id":"6a2bd6257a263366b3cfac26"}
  {"name":"apartment","entity_id":"6a2bd6937a263366b3cfaca7"}
  {"name":"rusted pier","entity_id":"6a2bd5d37a263366b3cfabb0"}
  {"name":"rusted pier","entity_id":"6a2bd5cd7a263366b3cfaba2"}
  ```
- **Known gap?** Cluster D1 — STILL OPEN

### [SEV: HIGH] Presence conflates recall with co-location (Cluster C — STILL OPEN)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9
- **Repro:** Multiple turns show incorrect presence.
- **Expected vs got:**
  - **Seq 11 (continue day after leaving Vex at pier):** `present_characters` shows `Vex` even though the narrative says the mercenary is alone in the apartment. Vex was left at the pier.
  - **Seq 12:** `present_characters` shows `Vex` again.
  - **Seq 16:** `present_characters` shows `Gray, the mercenary, the stranger` — "the mercenary" is the protagonist (should never be present in his own roster), and "the stranger" is a duplicate/alias of Gray.
  - **Seq 23:** `present_characters` shows `the bouncer, Bouncer, Jax` — "the bouncer" and "Bouncer" are the same entity, duplicated.
  - **Seq 29:** `present_characters` shows `Bouncer` — correct, only one.
- **Evidence:** Frame dumps from `agent-chat.ts` logs showing `present:` lines.
- **Known gap?** Cluster C — STILL OPEN

### [SEV: MED] Explicit-correction supersession did not fire (Cluster E — STILL OPEN)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9
- **Repro:** `"My sister's name is Mara."` (seq 12) → later `"My sister's name is Mira, not Mara."` (seq 30). After correction, query memories for `Mara` and `Mira`.
- **Expected vs got:** Old Mara atoms should be retired (`status: superseded`, `superseded_by_event_ids` stamped). Instead, all 3 Mara atoms remain `active` with `superseded_by_event_ids: null`. The new Mira atom (`6a2bd8db7a263366b3cfae23`) has `updates_memory_ids: ["6a2bd6367a263366b3cfac3f","6a2bd6647a263366b3cfac6d","6a2bd69c7a263366b3cfacb7"]` — forward links exist but backward marks are missing.
- **Evidence:**
  - Mara atoms: `status: active`, `superseded_by_event_ids: null`
  - Mira atom: `updates_memory_ids: [3 ids]`
- **Known gap?** Cluster E — STILL OPEN

---

## Other Findings

### [SEV: MED] Recap `when` field is null (Cluster G — STILL OPEN)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9
- **Repro:** `GET /chronicle/recap/6a2bd57cafc85d8941c370e9`
- **Expected vs got:** `when` should show the current story time. Instead: `{"when":null,"where":"the club","spine":"..."}`
- **Evidence:** `curl` response showing `when: null`
- **Known gap?** Cluster G — STILL OPEN

### [SEV: MED] Location cursor reset on `/continue day` (Cluster D3 — STILL OPEN)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9
- **Repro:** Seq 23: player at `the club`. Seq 24: `/continue day` → location jumps to `apartment` (the home base) without a travel event. The narrative says "The mercenary spends the interval in the shadows, the silence of the apartment pressing in..." but the player was at the club.
- **Expected vs got:** Continue should preserve the current location unless the narrative explicitly moves the character. Instead, the cursor reset to apartment.
- **Evidence:** `agent-chat.ts` logs: seq 23 `location: the club`, seq 24 `location: apartment` (no `travel` marker on seq 24).
- **Known gap?** Cluster D3 — STILL OPEN

### [SEV: LOW] Calendar API `month_names` field null despite correct `months` array (Cluster F2 — NEW / DATA SHAPE)
- **World/instance:** GM world "Neon Divide" iid=6a2bd57cafc85d8941c370e9
- **Repro:** `GET /chronicle/calendar/6a2bd57cafc85d8941c370e9`
- **Expected vs got:** `month_names` is `0`/`null`, but `months` array contains 12 Gregorian months (Jan–Dec). The frontend may not render months correctly if it reads `month_names`.
- **Evidence:** `{"calendars":[{"name":"Neon Divide Calendar","month_names":0}],"months":[{...January...},...]}`
- **Known gap?** NEW — data shape mismatch; `month_names` is empty but `months` is populated. The actual calendar is Gregorian (correct for cyber-noir).

### [SEV: LOW] Replay only works on latest turn (known limitation)
- **Repro:** `/replay 6a2bd7157a263366b3cfad23` (not latest) → `REPLAY_FAILED` with message "Replay is only available for the latest turn."
- **Expected vs got:** This is a known product constraint (J2), not a bug. The harness treats it as a terminal error and times out. The script should handle this gracefully.
- **Known gap?** J2 — known limitation

---

## Held GREEN

- **Structural audits:** `continuity-audit` 8/8 ✅, `audit:location` ✅, `audit:movement` 45/45 ✅, `audit:location-resolution` ✅, `audit:memory-links` 9/9 ✅, `rewind-audit` ✅
- **Rewind:** `POST /chronicle/rewind/:iid` with `{"sequence":25}` → deleted 6 events, 2 memories, max seq 24 ✅
- **Session bust:** `redis-cli del session:6a2bd57cafc85d8941c370e9` ✅
- **Timeline branch:** Fork + switch active + play on branch + switch back → no hang ✅
- **Event edit:** Wrong key 400, unchanged 400, changed 200 + recuration ✅
- **Memory edit:** `PUT /chronicle/memory/:id` → 200 ✅
- **Character edit:** `PUT /chronicle/character/:id` → 200 ✅
- **Flashback re-anchor:** `PUT /chronicle/calendar/event/:id/time-anchor` → works ✅
- **Calendar genre-fit:** Cyber-noir world has Gregorian months (Jan–Dec) ✅
- **Side-chat time freeze:** Story date + location cursor frozen during side-chat ✅
- **Side-chat scoped in dedicated thread:** `GET /chronicle/side-chats/:iid/:charId` shows correct 2-turn thread ✅

---

## Notes

- All structural/invariant audits passed while **semantic corruption** was present (side-chat leak, presence errors, location dupes). This matches the June 12 merged report observation.
- The `agent-chat.ts` harness is committed and was reused as-is.
- The server was already running with `localhost:3000` reachable.
- No worlds were deleted per the Lifecycle Policy.
- The Redis session cache was busted after the rewind test.
