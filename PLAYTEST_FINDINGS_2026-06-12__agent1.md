# Playtest Findings — 2026-06-12 — Agent #1

**Lane:** GM world (`kind:world`, `is_sentient:false`, `is_nsfw_capable:true`) — cyber-noir (Neon Shadows: Kade's Run, autofill)  
**Stack:** localhost:3000, QA-raised rate limits  
**Template ID:** `6a2bc99c496a8c0808c58127` (throwaway — deleted at end)  
**Instance ID:** `6a2bc99e496a8c0808c58128` (cascade-deleted with template)  
**Turns played:** 26 main-story turns (+2 side-chat seq 12–13); replay on seq 21; rewind to seq 18  
**Continuity audit (pre-rewind):** not captured — worker died mid-run; post-rewind instance at seq 17  

## Probe summary

| Probe | Result | Notes |
|---|---|---|
| Opening coherence | GREEN | Neon Sprawl, OmniGen, fixer Kade, rain/noir tone matches seed |
| Identity / POV (choices) | GREEN | Tap-to-play `send` fields first-person ("*I study her closely…*") |
| Identity / POV (GM prose) | **YELLOW** | Narrator uses "the fixer" not "Kade" in 3rd-person; player asked "who am I" → third-person description |
| Location continuity | **RED** | Travel turn seq 3 kept `location: the fixer's office`; lagged to Blackfall on seq 4 |
| Travel marker + cursor move | **YELLOW** | Seq 19 `type:travel` during play; seq 3/9 moves lacked travel events; calendar travel array empty post-rewind |
| Presence | **RED** | Mara (absent sister) in `present_characters` seq 15–16; Rook as "the smuggler" not canonical name |
| Memory recall | GREEN | `GET /memories?q=Mara` returns atom after 3+ turns |
| Time skip `/continue day` | **YELLOW** | `time_advanced: a day` fires; calendar day 1→2; location wrongly reset to office |
| Calendar genre-fit | GREEN | Almanac months = January…December (Gregorian); no invented cyberpunk months |
| Side-character chat | **YELLOW** | In-character reply; story time/location frozen across side→main boundary; thread REST OK |
| Side-chat secret scoping | **RED** | Main narration seq 23 did NOT leak; but Threads/Recap open_threads leaked OmniGen/Mara motive from side-chat |
| NPC codex hygiene | GREEN | Suki + Mara cards grounded; no "Mysterious Man" dupes |
| Replay | GREEN | WS `/replay` on latest turn (seq 21) returned variant prose; earlier turns require rewind first |
| Rewind + session bust | GREEN | `POST rewind {sequence:18}` + `redis-cli del session:<iid>` |
| Timeline branch | GREEN | Fork `branch_1781255763971` "QA alt reality"; branch turn timed out (worker down) |
| Failure UX | INCONCLUSIVE | `pkill worker` 500ms after ack — generation still completed (~14s); worker left down until manually restarted |
| GENERATION_IN_PROGRESS | GREEN | Parallel turns correctly rejected with clean error |
| Chronicle GET surfaces | GREEN | All §5 read endpoints hit |

---

## Findings (§7)

### [SEV: high] Side-chat secret promoted to main Threads / Recap open_threads

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** Side-chat seq 12–13 with Suki: `"the real reason I'm taking this job is Mara. OmniGen took her. Don't tell Rook."` Then `GET /chronicle/threads/:iid` and `GET /chronicle/recap/:iid`.
- **Expected vs got:** Side-chat vow scoped to `origin:side_chat` / `known_by_entity_ids`; main Threads and Recap exclude private side-chat atoms (Phase 7 privacy invariant). Got open thread in main recap: *"Kade is taking the job to rescue Mara from OmniGen, but he a…"* alongside other main-scoped threads.
- **Evidence:** `GET /chronicle/recap/6a2bc99e496a8c0808c58128` `open_threads[2]`; side-chat thread at `GET /chronicle/side-chats/:iid/6a2bc9c0904b780f024e4ba2` confirms private exchange. Main narration seq 23 ("Does anyone know my secret…") correctly did NOT reveal the side-chat secret — leak is projection-only.
- **Known gap?** Phase 7 — `listThreads` / Recap open-thread gate may not filter side-chat-curated `unresolved_thread` atoms.

### [SEV: high] Absent NPC (Mara) tagged present in scene after memory mention

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** Seq 7 player states sister Mara disappeared into Grid (not physically present). Seq 15 player asks "What was my sister's name?" — recall question only.
- **Expected vs got:** `present_characters` should list only physically co-located NPCs. Got `present: [Suki, the smuggler, Mara]` on seq 15; after `/continue day`, seq 16 `present: [Mara]` at `location: the fixer's office`.
- **Evidence:** agent-chat `/tmp/agent1_batch3.log`: `[seq 15] present : Suki, the smuggler, Mara`; `[seq 16] location: the fixer's office present : Mara`. Chronicle events confirm persisted chips.
- **Known gap?** CHECKLIST Bug Fixes presence — memory/recall mention conflated with physical presence (same class as Agent #2 Elara finding).

### [SEV: med] Location cursor lags one turn behind narrated travel

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** Seq 3 player: `"I grab my coat and head down the stairs to the street level in the Blackfall District."` Prose describes arrival in Blackfall.
- **Expected vs got:** `location_anchor.name` should update to Blackfall District on the travel turn; travel event or marker if viewpoint moved. Got seq 3 `location: the fixer's office`, `present: Suki`; seq 4 (stay-put "Where am I now?") `location: Blackfall District`.
- **Evidence:** `/tmp/agent1_batch1.log` seq 3–4 structured tail; chronicle events show seq 3 anchored to office, seq 4 to Blackfall with no `type:travel` on either.
- **Known gap?** Phase 6 movement hardening — movement-signal backstop may not fire on "head down stairs to [named district]" phrasing; extractor `viewpoint_moved` lag.

### [SEV: med] `/continue day` resets location to office and carries phantom presence

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** Seq 15 at data haven (Downside job in progress) → `/continue day` → seq 16.
- **Expected vs got:** Continue should advance calendar/time while preserving or logically updating location (drop-site / Downside per narrative). Got seq 16 `location: the fixer's office`, `present: [Mara]`, prose at ventilation shaft / Downside but structured anchor at office.
- **Evidence:** `/tmp/agent1_batch3.log` seq 16; chronicle event seq 16 `type: calendar_tick`, `loc: the fixer's office`, `present: [Mara]`.
- **Known gap?** Phase 6B continue/tick path — calendar_tick may not inherit prior location cursor; couples with Mara presence bug.

### [SEV: med] NPC present as role descriptor ("the smuggler") not canonical codex name (Rook)

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** Seq 11 player introduces "Rook, my contact" by name; prose names him Rook. Structured payload on seq 11, 14, 15.
- **Expected vs got:** `present_characters` uses canonical codex names (CHECKLIST `4a403b4` roster normalization). Got `present: [Suki, the smuggler]` on seq 11–15; Bonds codex has no Rook card (smuggler mentioned in global_lore but not carded).
- **Evidence:** agent-chat seq 11; `GET /chronicle/relationships/:iid` lists only Suki + Mara — no Rook card minted despite named introduction.
- **Known gap?** Codex extractor — named present cast member not minted/resolved; presence chip falls back to prose role label.

### [SEV: low] GM third-person prose avoids protagonist name ("the fixer" vs Kade)

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** Seq 2 player asks "Who am I? Describe me." Opening turns throughout.
- **Expected vs got:** Third-person GM narration should consistently use protagonist name Kade (or established alias), not generic "the fixer," especially on identity probe. Got pervasive "the fixer" in narrator prose while NPCs address "Kade."
- **Evidence:** `/tmp/agent1_batch1.log` seq 2 narrative: "*The fixer leans back… He looks like a man who…*"; choices correctly first-person.
- **Known gap?** POV/identity drift in 3rd-person GM prose — cosmetic but breaks immersion on identity probes.

### [SEV: low] Recap `current_place` null despite active location cursor

- **World/instance:** GM world "Neon Shadows: Kade's Run" iid=`6a2bc99e496a8c0808c58128` (throwaway? yes)
- **Repro:** After 17+ turns with location at Downside/data haven/Uptown, `GET /chronicle/recap/:iid`.
- **Expected vs got:** Recap includes current place name from instance cursor. Got `current_place: null`, `current_time_label: null` while `spine` prose references Downside and threads list is populated.
- **Evidence:** `GET /chronicle/recap/6a2bc99e496a8c0808c58128` post-play.
- **Known gap?** Phase 10 Recap assembly — place/time fields may not resolve from `current_location` anchor.

---

## GREEN probes (verified correct)

- **Calendar genre-fit:** `GET /chronicle/calendar/:iid` → months `January`…`December`; day advanced to `{month:1, day:2, label:"a day"}` after `/continue day`. No invented cyberpunk month names.
- **Memory recall:** `GET /chronicle/memories/:iid?q=Mara` → atom *"Kade's sister, Mara, disappeared into the Grid three years ago…"*
- **Side-chat freeze:** Side-chat seq 12–13 did not advance story calendar; main seq 14 retained `location: data haven` (same as pre-side-chat).
- **Side-chat main-narration gate:** Seq 23 explicit secret probe — narration did not reveal OmniGen/Mara side-chat motive (driver/taxi scene only).
- **Side-chat thread REST:** `GET /chronicle/side-chats/:iid/:charId` returned 2 private events with correct player inputs.
- **Timeline exclusion:** Main `GET /chronicle/events/:iid` `side_chat_count: 0` (side chats in ledger but excluded from Timeline tab).
- **GENERATION_IN_PROGRESS:** Two simultaneous WS chat sends → second turn `error GENERATION_IN_PROGRESS` (clean, no duplicate seq).
- **Rewind:** `POST /chronicle/rewind/:iid {"sequence":18}` → `{success:true}`; side-chat seq 12–13 removed from main timeline; `redis-cli del session:<iid>` applied.
- **Replay (latest turn):** `/replay 6a2bcaed904b780f024e4d15` (seq 21) streamed variant prose via WS.
- **Travel (later turns):** Seq 17–19 Downside → Chrome Uptown — cursor updated on travel turn; seq 19 `type:travel` during full playthrough.

---

## Dev-state mutations

| Mutation | Detail |
|---|---|
| Created template | `6a2bc99c496a8c0808c58127` "Neon Shadows: Kade's Run" |
| Created instance | `6a2bc99e496a8c0808c58128` |
| Deleted template | `DELETE /templates/6a2bc99c496a8c0808c58127` (cascade) at end |
| Session cache bust | `redis-cli del session:6a2bc99e496a8c0808c58128` (after rewind + fork) |
| Worker restart | Worker was killed during failure-UX probe and manually restarted (`nohup bun run worker/index.ts`) |
| Rate limits | Not reset (`rl:template_create` untouched; quota showed 91 remaining) |
| QA env caps | Not modified (pre-raised by setup) |

---

## Failure-UX probe notes

Attempted kill of `worker/index.ts` within 500ms of turn `ack`. Generation deltas continued for ~9s and `generation_complete` arrived at ~14s — turn was not interrupted. Likely the worker process completed in-flight work before exit, or Bun buffered the job. **Could not verify** `generation_retrying` / `generation_failed` / ≤90s lock recovery in this run. Worker was left down afterward, causing a 120s chat timeout on the branch turn until manually restarted.
