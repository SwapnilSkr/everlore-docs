# Playtest Findings — 2026-06-12 — Agent #2

**Lane:** GM world (`kind:world`, `is_sentient:false`, `is_nsfw_capable:false`) — high fantasy (Threads of Eldermarr, autofill)  
**Stack:** localhost:3000, QA-raised rate limits  
**Template ID:** `6a2bc994496a8c0808c58121` (throwaway — deleted at end)  
**Instance ID:** `6a2bc99c496a8c0808c58126` (cascade-deleted with template)  
**Turns played:** 26 main-story turns (+2 branch turns post-rewind); 2 side-chat turns (seq 24–25, removed by rewind)  
**Continuity audit (pre-rewind, seq 26):** 8 ok / 0 warn / 0 fail — `healthy: true`  
**Deterministic audits:** `audit:location`, `audit:movement`, `audit:location-resolution` — all green

## Probe summary

| Probe | Result | Notes |
|---|---|---|
| Opening coherence | GREEN | Frostwake, Lyra Vale, fate threads, Thornhaven — matches seed |
| Identity / POV | GREEN | Protagonist Lyra; choices first-person ("Follow the thread", etc.) |
| Location continuity | **RED** | Movement tracked in prose but atlas has 16 place nodes with duplicates |
| Presence | **YELLOW** | Corvin carry-forward on return GREEN; missing Mistress Elara marked present |
| Memory recall | GREEN | `q=Elara` finds 2 atoms after 3+ intervening turns |
| Time skip ("weeks pass") | GREEN | Seq 15 `time_advanced: weeks pass`, label set |
| `/continue season` | **YELLOW** | `time_advanced: a season` fires; calendar month jumps to Mistdeep while prose says Sunreach |
| Calendar genre-fit | GREEN | Almanac months = Frostwake / Sunreach / Emberfall / Mistdeep; weekdays themed; no Gregorian |
| Location graph (deeper/out/lateral) | **RED** | Deeper→archives OK; lateral watchtower mints duplicate; returns split Thornhaven/great hall |
| Fate threads | **YELLOW** | Thematic prose + flag `thread_of_destiny`; `fate_thread` field never populated on events |
| Side-character chat | **YELLOW** | In-character reply; story time frozen; thread REST OK; side-chat vow leaks to main Threads |
| NPC codex hygiene | GREEN | Master Corvin grounded; no "Mysterious Man" dupes |
| Replay + edit | **YELLOW** | Latest-turn WS replay works with choices; edit with `narrative` key silent no-op |
| Rewind + session bust | GREEN | `POST rewind {sequence:20}` deleted 7 events / 5 memories incl. side-chats; `del session:<iid>` |
| Timeline branch | GREEN | Fork `branch_1781255222789`; branch turn seq 20–21; switch back to main |
| Chronicle GET surfaces | GREEN | All §5 read endpoints hit |
| Continuity audit | GREEN (structural) | No warn/fail; location fragmentation not detected |

---

## Findings (§7)

### [SEV: high] Location atlas fragments — duplicate place entities break nested tree

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** 22-turn travel loop: Whispering Forest → mossy clearing → Thornhaven → great hall → lower archives → eastern watchtower (lateral) → Lyra's room → return great hall. Then `GET /chronicle/locations/:iid`.
- **Expected vs got:** One canonical node per place; returns reuse same entity; nested tree under one citadel root. Got **16 place entities** including duplicate names under different parents or orphan roots:
  - `Thornhaven` ×2 (one under `the city`, one orphan)
  - `the great hall` ×2 (under `the citadel` vs under orphan `Thornhaven`)
  - `mossy clearing` ×2 (one under forest, one orphan)
  - `eastern watchtower` ×2 (one under citadel with events, one orphan with 0 events)
  - `the citadel` ×2 (root orphan + child of itself)
  - Orphans: `great hall of Thornhaven`, `the mystical clearing`, `the city`
- **Evidence:** `GET /chronicle/locations/6a2bc99c496a8c0808c58126` places array; seq 20 travel to lateral watchtower minted orphan `6a2bcad0904b780f024e4ce9`, seq 21 reused `6a2bcaeb904b780f024e4d0c` under citadel.
- **Known gap?** LOCATION_GRAPH.md — variant naming + re-parent reveal not consolidating prior mints; related to open-limit #3 (roster cap) but fires early in single-citadel play.

### [SEV: med] Calendar desync after `/continue season` — prose Sunreach, almanac Mistdeep

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** Seq 15 player `"Weeks pass as I study the runes…"` → seq 16 `/continue season` → seq 17 `"What season is it now?"`.
- **Expected vs got:** After season skip into Sunreach (narrative: "height of Sunreach", "Zenith of the Verdant Eye"), almanac cursor should show month **Sunreach** (month 2). Got `story_calendar: {year:1, month:4, day:5, label:"a season"}` → month name **Mistdeep** per `GET /chronicle/calendar/:iid`.
- **Evidence:** Seq 16 `time_advanced: a season`; calendar endpoint `months[3].name === "Mistdeep"`; seq 17 narrative explicitly names Sunreach.
- **Known gap?** NEW — season advance amount vs themed month index; possible off-by-two season jump.

### [SEV: med] Side-chat secret promoted to main Threads / Recap open_threads

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** Side-chat seq 24–25 with Master Corvin: star-silver runes + `"I will not share this with the citadel guards."` Then `GET /chronicle/threads/:iid` and `GET /chronicle/recap/:iid` (pre-rewind).
- **Expected vs got:** Side-chat vow scoped to `origin:side_chat` / knowers only; main Threads excludes private side-chat atoms (Phase 7 privacy invariant). Got open thread `6a2bcb2d904b780f024e4d51`: *"Lyra vowed to keep Master Corvin's secrets from the citadel guards…"* in main `open_threads` (Recap + Threads tab).
- **Evidence:** `GET /chronicle/side-chats/:iid/6a2bca2a904b780f024e4c0e` shows the private exchange; `GET /chronicle/recap/:iid` lists the vow in `open_threads`.
- **Known gap?** Phase 7 — main-visible memory gate may not cover thread projection from side-chat curation.

### [SEV: med] Missing mentor Mistress Elara tagged present in scene

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** Turns 18–21 after Elara mentioned only in memory/recall (mentor is missing). Structured payload on seq 18, 19, 20, 21.
- **Expected vs got:** `present_characters` should not include absent/dead/missing NPCs unless they appear in prose. Got `present: Mistress Elara` on seq 18–21 while location is archives/watchtower (Elara not in scene).
- **Evidence:** agent-chat `[seq 18] present : Mistress Elara`; codex card exists with `disposition: remembered fondly`.
- **Known gap?** CHECKLIST Bug Fixes presence carry-forward — memory-triggered presence conflation.

### [SEV: med] Event edit with `narrative` field silent no-op (choices absent)

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** `PUT /chronicle/event/:eventId` body `{"narrative":"*Edited.* …"}` (playbook-style field).
- **Expected vs got:** Apply edit + regenerate choices/present, or 400. Got response with `choices: 0`, `present: 0`, no event id — same class as Agent #6 finding (schema expects `ai_response`).
- **Evidence:** PUT response immediately before template delete; Agent #6 confirmed pattern on character lane.
- **Known gap?** NEW (shared with Agent #6) — API ergonomics.

### [SEV: low] `fate_thread` metadata never populated on events

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** Full playthrough with fate-thread theme; inspect all events.
- **Expected vs got:** `generation_complete.event.fate_thread` / event doc field set when prose establishes a thread. Got `[]` — no event with non-null `fate_thread` across 21 events.
- **Evidence:** `GET /chronicle/events/:iid` map over `.fate_thread`.
- **Known gap?** NEW — extractor may not emit fate_thread for GM worlds.

### [SEV: low] Recap `when` null despite active calendar cursor

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** After season skip, `GET /chronicle/recap/:iid`.
- **Expected vs got:** Recap includes current story time (themed season + day). Got `where: "the great hall"` populated but `when: null` while calendar endpoint shows `{month:4, day:5, label:"a season"}`.
- **Evidence:** Recap JSON keys `where` / `when`; calendar `current_time_anchor`.
- **Known gap?** Phase 10 Recap — time projection gap.

### [SEV: low] Sealed vault not distinct atlas node (stays "lower archives")

- **World/instance:** GM world "Threads of Eldermarr" iid=`6a2bc99c496a8c0808c58126` (throwaway? yes)
- **Repro:** Seq 13 `"I push deeper into the archives until I find a vault sealed with star-silver runes."`
- **Expected vs got:** Deeper movement mints child place under archives (vault / sealed chamber). Got `location_anchor.name: the citadel's lower archives` unchanged; no vault node in atlas.
- **Evidence:** Seq 13 structured tail; locations list has archives with 7 events, no vault child.
- **Known gap?** Location Graph cartographer — `movement:deeper` under-reports for named sub-rooms.

---

## GREEN (verified, no new ticket)

- **Themed calendar:** Frostwake / Sunreach / Emberfall / Mistdeep + Star's Day / Moon's Day / Witch's Day — no January–December.
- **Time skips:** `"Weeks pass…"` → seq 15 `time_advanced: weeks pass`; `/continue season` → seq 16 `time_advanced: a season`; `/continue day` on branch → seq 21 `time_advanced: a day`.
- **Possessive room:** `"I go to my private quarters"` → `Lyra's room` under citadel, parents correct.
- **Side-chat mechanics:** seq 24–25 streamed in-character; calendar cursor unchanged during side chats; per-character thread endpoint populated.
- **Side-chat main narration:** Seq 26 confrontation did not verbatim leak star-silver side-chat content in prose (thread projection leak is separate).
- **Replay:** `/replay` on latest turn returned variant with 4 choices; earlier-turn replay correctly rejected.
- **Rewind:** `{sequence:20}` removed side-chats + later events; session cache busted.
- **Timeline fork:** Branch created and accepted a turn without polluting main timeline events.
- **Travel markers:** Travel events at seq 7, 14, 20, 22, 23, 26 with plausible from/to.
- **Memory recall:** Elara lie/threads teaching retrievable via Echoes search.
- **Master Corvin presence:** Returned to great hall seq 14 — Corvin still present after archives detour.

---

## Dev-state mutations

| Action | Detail |
|---|---|
| Created template | `6a2bc994496a8c0808c58121` |
| Created instance | `6a2bc99c496a8c0808c58126` |
| Rewind | `{sequence:20}` — deleted 7 events, 5 memories |
| Session bust | `redis-cli del session:6a2bc99c496a8c0808c58126` (after rewind + post-edit attempt) |
| Timeline fork | `branch_1781255222789` |
| Cleanup | `DELETE /templates/6a2bc994496a8c0808c58121` — cascade removed instance |

No rate-limit resets, no server config edits, no keeper worlds touched.
