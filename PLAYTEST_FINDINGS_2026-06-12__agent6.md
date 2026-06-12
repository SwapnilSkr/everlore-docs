# Playtest Findings — 2026-06-12 — Agent #6

**Lane:** Character companion (`kind:character`, `is_sentient:true`, `is_nsfw_capable:false`) — fantasy adventurer (Elara Thornwood, autofill)  
**Stack:** localhost:3000, QA-raised rate limits  
**Template ID:** `6a2bc9a4496a8c0808c58130` (throwaway — deleted at end)  
**Instance ID:** `6a2bc9a7496a8c0808c58131` (cascade-deleted with template)  
**Turns played:** 19 main-story turns (+1 post-rewind); 0 side-chats  
**Continuity audit (pre-rewind, seq 19):** 8 ok / 0 warn / 0 fail — `healthy: true`  
**Continuity audit (post-rewind, seq 12):** 8 ok / 0 warn / 0 fail — `healthy: true`

## Probe summary

| Probe | Result | Notes |
|---|---|---|
| Opening coherence | GREEN | Elara in-voice; fantasy tone; companion addresses player as hired reckless soul |
| Identity / POV (character lane) | **RED** | Memories invert player facts onto Elara; recap threads attribute Mara's locket to Elara |
| Location continuity | **YELLOW** | Movement to Grayreach/keep/gatehouse tracked; duplicate place entities (see below) |
| Presence | GREEN | `present_characters` consistently Elara only on main turns |
| Memory recall | GREEN | Seq 7 correctly recalled Mara + locket after 2 intervening turns |
| Time skip (`/continue day`) | GREEN | Seq 10 `time_advanced: a day`; calendar day 1→2 |
| Calendar genre-fit | **YELLOW** | Themed calendar in Almanac (Thornbloom, etc.) — correct for fantasy; prose invented "Month of the Pale Frost" / "Frost-Moon" not in calendar |
| Side-character chat | SKIP | No secondary `character_id` besides Elara; lane is 1:1 companion |
| NPC codex hygiene | **RED** | Vague passers-by minted Bonds cards; player-relative Mara minted as NPC |
| Replay + edit | **YELLOW** | Latest WS replay works; choices present on event; edit with wrong field silently no-ops |
| Rewind + session bust | GREEN | `POST rewind {sequence:12}` deleted 8 events / 10 memories; post-rewind turn coherent |
| Timeline branch | GREEN | Fork created `branch_1781255144729` |
| Chronicle GET surfaces | GREEN | All §5 read endpoints hit; side-chats empty as expected |
| Continuity audit | GREEN (structural) | No warn/fail; semantic corruption not detected |

---

## Findings (§7)

### [SEV: high] Player facts projected onto companion — Mara locket attributed to Elara

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** Turn 5: `"I tell you quietly: my sister's name is Mara, and she gave me this silver locket before I left home."` Then read memories/threads/recap.
- **Expected vs got:** Player's sister + locket should stay player-scoped. Got memory atom `6a2bc9ea904b780f024e4bd6`: *"Elara Thornwood revealed that **her sister's name is Mara**, and that Mara gave **her** a silver locket before she left home"*. Thread `6a2bcae4904b780f024e4d00` (post-rewind still present in earlier state): grief tied to Mara attributed to Elara. Seq 14 memory: *"amusement and concern toward **Mara**"* when player asked presence question (player conflated with Mara).
- **Evidence:** `GET /chronicle/memories/:iid?q=Mara`; `GET /chronicle/threads/:iid`; event seq 5 `codex_deltas` correctly says protagonist's sister but memory curation inverts.
- **Known gap?** NEW — character-lane identity boundary (player vs sentient companion) not enforced in memory projection.

### [SEV: high] Event edit accepts `narrative` field — silent no-op, still marks edited + re-curates

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** `PUT /chronicle/event/6a2bc9e5904b780f024e4bcf` body `{"narrative":"*Elara pauses at the charm-seller stall..."}` (playbook-style field name).
- **Expected vs got:** Either apply edit or 400. Got `{success, recuration_queued, memories_deleted, choices}` with `is_user_edited:true` but `data.ai_response` **unchanged** (still original prose). Schema expects `ai_response` per `EditEventBody`.
- **Evidence:** `GET /chronicle/events/:iid` seq 5 after edit; `edit_history` appended twice with identical `previous_data`.
- **Known gap?** NEW — API ergonomics / silent discard of wrong key.

### [SEV: med] Vague passers-by mint full Bonds codex cards

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** Turn 4: hooded charm-seller — *"I don't ask their name, just glance at their wares."* Turn 11: old woman at ford — *"I don't speak to her, just note she's there."*
- **Expected vs got:** No named interaction → no durable codex card (or single deferred stub). Got Bonds entries **Hooded Figure** (`6a2bc9e0904b780f024e4bca`) and **Old Woman** (`6a2bca67904b780f024e4c79`) with `role: non-player character`, `disposition: unknown`.
- **Evidence:** `GET /chronicle/relationships/:iid`; recap `bonds` array; `[codex] updated` frames on turns 4 and 11.
- **Known gap?** Relates to HANDOFF duplicate-character / extractor-drift hardening — vague descriptors still promote to cards.

### [SEV: med] Player off-screen relative Mara promoted to Bonds NPC card

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** Turn 5 (player sister Mara, never present in scene).
- **Expected vs got:** Mara should live in player memory atoms only, not Bonds tab beside Elara. Got codex card `Mara` / `role: sister` / `disposition: not applicable` in `GET /chronicle/relationships/:iid` and recap bonds.
- **Evidence:** Event seq 5 `codex_deltas` includes Mara; relationships endpoint lists 3 NPCs (Mara, Hooded Figure, Old Woman) plus Elara protagonist off-list.
- **Known gap?** NEW — codex promotion rules for player-mentioned but absent characters.

### [SEV: med] Replay variant swaps POV — player narrated as Elara

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** WS `/replay 6a2bcaf6904b780f024e4d24` (seq 19 — player asked Elara about calendar from battlements).
- **Expected vs got:** Companion continues speaking to player in second person. Got replay prose in **player-voice**: *"you haul yourself over the crenelated lip… you keep the ironbark bow gripped tight"* — player holds Elara's bow.
- **Evidence:** `/tmp/agent6_play_batch4.log` replay_complete narrative; recap `spine` overwritten with same POV-swapped variant.
- **Known gap?** NEW — replay generation ignores `is_sentient` companion POV contract.

### [SEV: med] Duplicate place entities in location atlas

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** Travel Wilds → Grayreach → gatehouse → keep over turns 2–19.
- **Expected vs got:** One canonical node per place. Got parallel roots: `Verdant Wilds` + `the Verdant Wilds` + `the Wilds` (0 events); two `the keep` nodes (parent/child, both `place_kind: null`).
- **Evidence:** `GET /chronicle/locations/:iid` places array (7 entries, 3 for same forest, 2 for keep).
- **Known gap?** LOCATION_GRAPH.md open-limit #1 (same-name collision) — variant naming ("the X" vs "X") not deduped.

### [SEV: med] Calendar prose invents month names absent from Almanac

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** Turn 19: `"what calendar month is it in this realm?"`
- **Expected vs got:** Themed months from calendar (`Thornbloom`, `Gloomshade`, …). Got *"Month of the Pale Frost"* and *"Frost-Moon"* — not in `GET /chronicle/calendar/:iid` month list.
- **Evidence:** Seq 19 narrative in `/tmp/agent6_play_batch3.log`; calendar `months[].name` list.
- **Known gap?** AUTOCHAT_PLAYBOOK §4 step 7 (calendar genre-fit) — themed calendar exists but generation doesn't ground in it.

### [SEV: low] WS replay of non-latest turn fails; harness can hang

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** `/replay 6a2bc9d9904b780f024e4bc2` (seq 4) mid-batch after seq 19 completed.
- **Expected vs got:** Playbook implies replay any eventId. Got `REPLAY_FAILED: Replay is only available for the latest turn. Rewind first for earlier turns.` — batch3 **120s TIMEOUT** waiting for completion.
- **Evidence:** `/tmp/agent6_play_batch3.log` lines 72–85.
- **Known gap?** Product constraint — document in playbook; harness should treat REPLAY_FAILED as terminal for that step.

### [SEV: low] REST `POST /chronicle/replay/:eventId` returns empty after rewind

- **World/instance:** character "Elara Thornwood" iid=`6a2bc9a7496a8c0808c58131` (throwaway? yes)
- **Repro:** After rewind to seq 12, `POST /chronicle/replay/6a2bca72904b780f024e4c83`.
- **Expected vs got:** New variant(s) with choices. Got `{variants: [], has_choices: []}` (empty body).
- **Evidence:** curl output in agent6 session; latest turn was seq 11 after rewind so seq 12 event no longer latest.
- **Known gap?** NEW — REST replay may require latest-event invariant same as WS.

---

## Chronicle surfaces exercised

**GET:** recap, events, memories (`?q=Mara`), calendar, threads, relationships, relationships/:charId/memories (404 after rewind for Mara id), locations, locations/:entityId, side-chats  
**PUT/POST/DELETE:** event edit, memory edit, character edit, replay (WS + REST), rewind `{sequence:12}`, timeline fork, session bust `redis-cli del session:6a2bc9a7496a8c0808c58131`

## Cleanup

- `DELETE /templates/6a2bc9a4496a8c0808c58130` → `{deleted: true}` (instances cascade)
- Session key deleted after rewind (redis returned 0 — key may not have existed)
- No rate-limit or server config edits; no `rl:template_create` reset

## Dev-state mutations log

| Mutation | Value |
|---|---|
| Created template | `6a2bc9a4496a8c0808c58130` |
| Created instance | `6a2bc9a7496a8c0808c58131` |
| Deleted template | yes (end of run) |
| Rewind | to sequence 12 |
| Timeline fork | `branch_1781255144729` |
| Session bust | `session:6a2bc9a7496a8c0808c58131` |
