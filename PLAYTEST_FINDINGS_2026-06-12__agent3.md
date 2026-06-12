# Playtest Findings — Agent 3 (Sentient / Modern)

**Date:** 2026-06-12  
**Lane:** Sentient world (`is_sentient:true`, `is_nsfw_capable:false`), modern city-AI theme  
**Template:** `6a2bc9e5496a8c0808c58136` — "Meridian Transit" (deleted after run)  
**Instance:** `6a2bc9f9496a8c0808c58147` (cascade-deleted with template)  
**Turns played:** 22 main + 2 side-chat + 1 replay + 1 post-rewind (≥20 requirement met)  
**Token:** shared dev bearer (not logged)

---

## Probe summary

| Probe | Result |
|---|---|
| Opening coherence (sentient AI as interlocutor) | GREEN |
| Player POV / choices phrasing | GREEN |
| Live chat memory recall (badge L-4472) | GREEN |
| Location anchor carry-forward | GREEN |
| `/continue day` + time_advanced | GREEN |
| Side-chat secret isolation (UrbanCore favor) | GREEN |
| Side-chat thread in Chronicle | GREEN |
| Replay via `/replay <event._id>` | GREEN |
| Rewind via `POST …/rewind {sequence:N}` | GREEN |
| Timeline fork + switch active | GREEN (fork created; branch turn hung — see below) |
| Memory edit `PUT /chronicle/memory/:id` | GREEN |
| Continuity audit | GREEN (8/8 ok at max seq 14 post-rewind) |
| Deterministic audits (`audit:movement`, `audit:location-resolution`, `audit:memory-links`, `rewind-audit`) | GREEN |
| Memory atom attribution (player vs Meridian) | **FAIL** |
| NPC presence (`present_characters`) | **FAIL** |
| Almanac Gregorian / story calendar | **FAIL** |
| Recap / thread projection POV | **FAIL** |
| Branch timeline turn generation | **FAIL** (timeout) |

---

## Findings

### [SEV: high] Memory extractor swaps player facts onto the sentient world character

- **World/instance:** sentient world "Meridian Transit" iid=`6a2bc9f9496a8c0808c58147` (throwaway? yes)
- **Repro:** Turn 6 — player states *"my liaison badge number is L-4472 and my supervisor is Director Chen."* Play 3 unrelated turns, then `GET /chronicle/memories/:iid?q=L-4472`.
- **Expected vs got:** Memory should read *"The municipal liaison's badge is L-4472; supervisor Director Chen."* Got: `"Meridian's liaison badge number is L-4472, and her supervisor is Director Chen."` — badge and supervisor attributed to Meridian (she/her), not the player liaison.
- **Evidence:** Memory atom `6a2bcab7904b780f024e4ccc`; subjects=`["Meridian"]`; subject_entity_ids point at Meridian's entity. After replay, a new atom appeared: *"Officer Reyes confirmed that Meridian's badge number is L-4472"* (`6a2bcc71904b780f024e4db8`).
- **Known gap?** NEW — sentient-world POV inversion in memory extraction; corrupts Echoes tab and RAG for any player-stated self-facts.

### [SEV: med] Open threads / recap misattribute player movement and role in sentient worlds

- **World/instance:** sentient world "Meridian Transit" iid=`6a2bc9f9496a8c0808c58147` (throwaway? yes)
- **Repro:** Player walks to control room (turn 4), asks about Civic Center package (turn 8). Then `GET /chronicle/threads/:iid` and `GET /chronicle/recap/:iid`.
- **Expected vs got:** Threads should describe the **player liaison** visiting places and asking Meridian. Got threads like *"Meridian entered the Central Transit Hub control room"* and *"The operator in the control room asked Meridian if they could address the suspicious package"* — player movement folded onto Meridian / vague "operator."
- **Evidence:** Thread ids `6a2bca51904b780f024e4c50`, `6a2bca53904b780f024e4c54`; recap `when` field is `null` despite `where: "Deep Core"`.
- **Known gap?** Related to sentient-world protagonist modeling (template stores Meridian as `protagonist`); projection layer treats the world-AI as the story subject.

### [SEV: med] Introduced NPC never appears in `present_characters` at scene location

- **World/instance:** sentient world "Meridian Transit" iid=`6a2bc9f9496a8c0808c58147` (throwaway? yes)
- **Repro:** Turn 13 — *"I spot Officer Reyes near the cordoned package and walk over to talk to her."* Turn 14 — *"Who is present with me right now?"* Narrative names Reyes; check `generation_complete.present_characters`.
- **Expected vs got:** `present_characters` should include Officer Reyes (`6a2bcaed904b780f024e4d13`) at Civic Center. Got: `present : Meridian` only on every turn (22/22 main turns); one `/continue day` frame showed `present : -`.
- **Evidence:** `/tmp/agent3_play_batch2.log` seq 13–14; Bonds ledger correctly minted Officer Reyes card; side-chat with her worked.
- **Known gap?** HANDOFF P2.6 movement/presence — partial; codex mints NPC but presence ledger omits on-scene humans when sentient entity dominates.

### [SEV: med] Modern Almanac lacks Gregorian month names; story calendar stays abstract

- **World/instance:** sentient world "Meridian Transit" iid=`6a2bc9f9496a8c0808c58147` (throwaway? yes)
- **Repro:** Seed specifies 2024 / Gregorian. After `/continue day`, ask *"what month and year?"* (narrative answers "2024"), then `GET /chronicle/calendar/:iid`.
- **Expected vs got:** Almanac should expose Gregorian month names (January…December) and anchor near June 2024. Got: `calendars[0].month_names` is `null` (length 0); `story_calendar` stays `{year:1, month:1, day:2, label:"a day"}` with no month name; recap `when` is null.
- **Evidence:** Calendar API response; narrative seq 16 correctly says "2024" and "Gregorian system" in prose only.
- **Known gap?** CHECKLIST/HANDOFF "Calendar genre-fit (June 12)" — narrative fixed, Almanac API still broken for modern worlds.

### [SEV: med] Branch timeline turn hangs (120s timeout) after switching active timeline

- **World/instance:** sentient world "Meridian Transit" iid=`6a2bc9f9496a8c0808c58147` (throwaway? yes)
- **Repro:** `POST /chronicle/calendar/:iid/timeline` `{name:"QA branch — bomb squad arrives", fork_at_sequence:15}` → `PUT …/timeline/active` `{timeline_id:"branch_1781255380245"}` → `redis-cli del session:<iid>` → `agent-chat.ts` one chat turn.
- **Expected vs got:** Turn completes on branch. Got: 120s timeout, no `generation_complete`; main timeline turn succeeds immediately after switching back to `main`.
- **Evidence:** `/tmp/agent3_branch_turn.log`; calendar showed `active_timeline_id: null` between fork and switch-back.
- **Known gap?** NEW — branch play path may not dispatch generation when active timeline ≠ main.

### [SEV: low] Spurious `REPLAY_FAILED` WS errors during unrelated turns

- **World/instance:** sentient world "Meridian Transit" iid=`6a2bc9f9496a8c0808c58147` (throwaway? yes)
- **Repro:** During batch 2, `/continue day` (seq 15) and turn 14 (`Who is present…`) — no `/replay` sent.
- **Expected vs got:** No replay errors. Got: `!! ERROR REPLAY_FAILED Replay is only available for the latest turn` on WS (harness instance-filtered).
- **Evidence:** `/tmp/agent3_play_batch2.log` lines around seq 14–15 continue.
- **Known gap?** Possibly parallel-agent cross-talk or stale replay job; worth confirming not emitted to player UI on single-agent runs.

### [SEV: low] Playbook rewind body field mismatch

- **World/instance:** n/a (docs)
- **Repro:** `POST /chronicle/rewind/:iid` with `{to_sequence:18}` per AUTOCHAT_PLAYBOOK §4 step 11.
- **Expected vs got:** Rewind succeeds. Got: validation error — expects `{sequence: number}`.
- **Evidence:** `{to_sequence:18}` → 422; `{sequence:15}` → `{success:true, deletedEvents:9, deletedMemories:9}`.
- **Known gap?** Doc drift — update playbook to `{sequence}`.

---

## Chronicle surfaces exercised

**GET:** recap, events, memories (search `q=L-4472`, `q=badge`), calendar, threads, relationships, relationships/:char/memories, locations, side-chats, side-chats/:charId  

**POST/PUT:** replay (REST + WS), rewind, calendar timeline fork, timeline active switch, memory edit, event edit (partial — response `{success:true}` on memory; event edit sent)

**Audits:** `audit:location`, `audit:movement`, `audit:location-resolution`, `audit:memory-links`, `scripts/rewind-audit.ts`, `GET /admin/instances/:iid/continuity-audit`

---

## Cleanup

- `DELETE /templates/6a2bc9e5496a8c0808c58136` → `{deleted:true}` (instance cascade-deleted)
- `redis-cli del session:6a2bc9f9496a8c0808c58147` run after rewind, timeline switch, memory edit
- No rate-limit or server config changes made
- Logs: `/tmp/agent3_play_batch{1,2,3}.log`, `/tmp/agent3_replay2.log`, `/tmp/agent3_branch_turn.log`, `/tmp/agent3_post_rewind.log`
