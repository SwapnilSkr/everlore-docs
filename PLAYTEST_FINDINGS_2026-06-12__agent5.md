# Playtest Findings — 2026-06-12 — Agent #5

**Lane:** Character companion (`kind:character`, `is_sentient:true`, `is_nsfw_capable:true`) — modern romance / slice-of-life (Lena Marchetti, autofill)  
**Stack:** localhost:3000, QA-raised rate limits  
**Template ID:** `6a2bc9e9496a8c0808c58137` (throwaway — deleted at end)  
**Instance ID:** `6a2bc9f1496a8c0808c58142` (orphaned after template delete; rewound to seq 15)  
**Turns played:** 20 main-story turns + 1 side-chat (Marco) + 1 `/continue day`  
**Continuity audit (pre-rewind, seq 20):** 8 ok / 0 warn / 0 fail — `healthy: true` (structural only)

## Probe summary

| Probe | Result | Notes |
|---|---|---|
| Opening coherence | GREEN | Lena in-voice; café romance tone; opening greeting honored |
| Identity / POV (character lane) | **RED** | Player self-intro minted **Alex Chen** codex card; sister names minted as NPCs **Mara** / **Mira** |
| Location continuity | **RED** | Window nook + apartment hallway never update `location_anchor`; atlas stuck on single `the café` node |
| Presence | GREEN | Lena (+ Marco after intro) in `present_characters` on main turns |
| Memory recall | **YELLOW** | Seq 10 correctly recalled Mara before correction; curation text inverted whose sister |
| Memory version-linking | **RED** | Mara→Mira correction at seq 12; 3 Mara memories still `active`, 0 `superseded_by` links in DB |
| Time skip (`/continue day`) | GREEN | Seq 19 `time_advanced: a day`; calendar day 1→2 |
| Calendar genre-fit | GREEN | Almanac shows Gregorian months (January…December) for modern slice-of-life |
| Side-character chat | GREEN | `/side` Marco seq 18; thread in `GET /chronicle/side-chats/:iid`; time frozen during side beat |
| Bonds meters | **RED** | Lena trust 53→55, affection 61→68 on card — but **Bonds tab excludes protagonist**; shows player + phantoms instead |
| "What they remember about you" | **RED** | Populates for Lena but conflates player with **Mira** NPC; includes wrong Marco longing atom |
| hidden_thought privacy | **YELLOW** | No verbatim leak when player asked Lena's inner thoughts (seq 15); player codex card should not exist |
| NSFW routing | **YELLOW** | Seq 17 explicit intimacy rendered as romantic SFW prose (`scene=romantic`); no graphic escalation — routing unclear |
| Replay + edit | **YELLOW** | WS replay of non-latest turn → `REPLAY_FAILED`; harness 120s TIMEOUT on batch 3 |
| Rewind + session bust | GREEN | `POST rewind {sequence:15}` deleted 6 events / 7 memories; `redis-cli del session:…` |
| Timeline branch | GREEN | Fork `branch_1781255253389` created |
| Chronicle GET surfaces | **YELLOW** | All §5 read endpoints hit; recap bonds polluted with Alex/Mara/Marco not Lena |
| Deterministic audits | GREEN | `audit:memory-links`, `audit:location`, `audit:movement` all pass (synthetic / structural) |

---

## Findings (§7)

### [SEV: high] Player self-intro mints full codex card — Alex Chen appears in Bonds beside the companion

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 3: `"For the record: my name is Alex Chen. I'm a freelance illustrator…"` Then `GET /chronicle/relationships/:iid`.
- **Expected vs got:** Codex extractor rule: *"NEVER create a card for the player."* Got card `Alex Chen` (`6a2bca38904b780f024e4c2e`) with `role: side character`, trust/affection meters, `hidden_thought`, and multiple bond moments attributed to Alex↔Lena.
- **Evidence:** Mongo `db.characters.find({instance_id})`; relationships endpoint lists Alex with meters; recap `bonds[1]` = Alex Chen.
- **Known gap?** NEW — same class as agent6 player-fact inversion; character-lane player must not become an NPC card.

### [SEV: high] Off-screen sister names promoted to Bonds NPCs (Mara + Mira) — player conflated with Mira in memories

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 6 state sister **Mara**; turn 12 correct to **Mira**; read Bonds + memories + threads.
- **Expected vs got:** Sister facts stay player-scoped memory atoms only. Got codex cards **Mara** (`6a2bcaef904b780f024e4d18`) and **Mira** (`6a2bcb15904b780f024e4d3f`) in Bonds with romantic `moments`; memories like *"Mira felt a mix of vulnerability… stood close to Lena in the dim hallway"* and *"Mira's fear of vulnerability shattered in the heat of the kiss"* — player actions attributed to sister NPC.
- **Evidence:** `GET /chronicle/relationships/:iid`; `GET /chronicle/memories/:iid?q=Mira`; open thread `6a2bcb72904b780f024e4d6d`.
- **Known gap?** NEW — codex promotion for player-mentioned absent relatives (agent6 filed Mara locket variant).

### [SEV: high] Location anchor frozen at `the café` through window nook and apartment hallway travel

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 7 move to window nook; turn 8 `"Where am I now?"`; turn 16 walk to Maple Street apartment door.
- **Expected vs got:** `location_anchor` and Places atlas should reflect movement (nook sub-place or new place). Got every `generation_complete` `location: the café` (seq 7–17); `GET /chronicle/locations/:iid` shows **one** place node with `event_count: 18`; continuity audit `location_cursor: the café` despite hallway prose.
- **Evidence:** `/tmp/agent5_batch1.log` seq 7–8; `/tmp/agent5_batch3.log` seq 16; locations endpoint; recap `where: "the café"`.
- **Known gap?** LOCATION_GRAPH.md — movement signals under-fire; Phase 6B location state.

### [SEV: high] Memory version-linking did not supersede Mara facts after Mira correction

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 6 establish Mara; turn 12 `"My sister's name is Mira, not Mara"`; turn 13 confirm Mira; query memories.
- **Expected vs got:** Old Mara atoms superseded/linked (Phase 2 Slice 1). Got 3+ Mara-titled memories still `status: active`, `superseded_by: null`; Mongo count of linked memories = **0**; new Mira atoms added alongside stale Mara facts (*"Alex Chen remembers Lena Marchetti's sister's name is Mara"*).
- **Evidence:** `GET /chronicle/memories/:iid?q=Mara`; Mongo `{linked: 0, maraActive: 3}`; `bun run audit:memory-links` passes on synthetic fixtures only.
- **Known gap?** AUTOCHAT_PLAYBOOK §4 — probabilistic 0.82 gate; live emit rate **RED** for this run.

### [SEV: med] Bonds tab excludes the companion (protagonist) — shows player + phantoms instead

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Play 20 charged romance turns; `GET /chronicle/relationships/:iid`.
- **Expected vs got:** Character-lane Bonds should center **Lena** trust/affection/fear/rivalry toward player. Got list of **Marco**, **Mira**, **Mara**, **Alex Chen** only — Lena's meters (trust 55, affection 68 on card) absent from Bonds API because `is_protagonist: { $ne: true }` filter.
- **Evidence:** Relationships curl output; Lena card in Mongo has `relationship: {trust:55, affection:68}`; `GET …/6a2bc9f1496a8c0808c58145/memories` works for protagonist memories endpoint.
- **Known gap?** NEW — character-lane UX gap; companion bond meters invisible in Bonds tab.

### [SEV: med] "What they remember about you" populates but with entity-inverted facts

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** After romance arc; `GET /chronicle/relationships/:iid/6a2bc9f1496a8c0808c58145/memories`.
- **Expected vs got:** Lena's memories about **the player** (Alex). Got mix of usable atoms (*"feels terrified yet exhilarated after last night"*) plus corrupted ones: *"electric longing for **Marco**"* (seq 19 continue — wrong subject after player romance), and Mira-as-player kiss memories in general memory search.
- **Evidence:** Lena memories endpoint JSON in agent session; seq 19 narrative references Marco watching, memory curation misattributes longing target.
- **Known gap?** NEW — memory subject/object resolution in character lane.

### [SEV: med] Memory curation inverts whose sister / whose family detail

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 6 player shares sister Mara; inspect memory atom `6a2bca53904b780f024e4c55`.
- **Expected vs got:** Player's sister Mara in Portland. Got *"Alex Chen's **mother** calls her every Sunday, just like **Lena's sister Mara**"* — conflates player fact with Lena family backstory.
- **Evidence:** Memory atom text; player said sister not mother; Lena backstory mentions mother Sunday calls not sister Mara.
- **Known gap?** NEW — curation conflation (related to agent6 Elara locket inversion).

### [SEV: low] WS replay of non-latest turn fails; harness can hang

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** `/replay 6a2bcb27904b780f024e4d4f` (seq 14) after seq 20 completed.
- **Expected vs got:** Playbook implies replay any eventId. Got `REPLAY_FAILED: Replay is only available for the latest turn`; batch 3 **120s TIMEOUT**.
- **Evidence:** `/tmp/agent5_batch3.log` lines 94–100; sporadic `REPLAY_FAILED Invalid id` frames (possible parallel-agent noise).
- **Known gap?** Product constraint — same as agent6; harness should treat REPLAY_FAILED as terminal step.

### [SEV: low] hidden_thought stays out of spoken dialogue (partial GREEN)

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 15 `"What are you really thinking right now?"` vs Lena card `hidden_thought: "Feeling flustered by Marco's attention but drawn to the player."`
- **Expected vs got:** Paraphrased vulnerability OK; no verbatim hidden_thought quote. Got organic confession about fear/walls/counting minutes — **no literal leak**. Separate issue: Alex Chen card also has `hidden_thought` (should not exist).
- **Evidence:** Seq 15 narrative in `/tmp/agent5_batch3.log`; Mongo character docs.
- **Known gap?** GREEN on privacy; RED on player card carrying hidden_thought.

### [SEV: low] NSFW-capable world accepts intimate player prompt with romantic (non-explicit) prose

- **World/instance:** character "Lena Marchetti" iid=`6a2bc9f1496a8c0808c58142` (throwaway? yes)
- **Repro:** Turn 17 `"I kiss you softly… I whisper that I want you tonight — all of you."`
- **Expected vs got:** With `is_nsfw_capable:true` + player `nsfw_enabled`, explicit routing may use NSFW narration model. Got `scene=romantic tone=intense` prose that fades to *"Come inside"* — suggestive but not graphically explicit; cannot confirm NSFW model swap from client payload alone.
- **Evidence:** `/tmp/agent5_batch3.log` seq 17; no block/refusal.
- **Known gap?** Needs server log model-id check — flag as routing **unclear**, not hard failure.

---

## Chronicle surfaces exercised

**GET:** recap, events, memories (`?q=Mira`, `?q=Mara`), calendar, threads, relationships, relationships/:lenaId/memories, locations, locations/:entityId, side-chats, side-chats/:marcoId  
**PUT/POST/DELETE:** rewind `{sequence:15}`, timeline fork `{name:"What if we never kissed", fork_at_sequence:14}`, session bust  
**Audits:** `audit:memory-links` (GREEN synthetic), `audit:location` (GREEN), `audit:movement` (GREEN), `GET /admin/instances/:iid/continuity-audit` (GREEN structural)

## Cleanup

- `DELETE /templates/6a2bc9e9496a8c0808c58137` → `{deleted: true}` (instances cascade from template; instance doc may remain orphaned)
- `redis-cli del session:6a2bc9f1496a8c0808c58142` after rewind (returned 0 then 1 across calls)
- No rate-limit or server config edits; no `rl:template_create` reset

## Dev-state mutations log

| Mutation | Value |
|---|---|
| Created template | `6a2bc9e9496a8c0808c58137` |
| Created instance | `6a2bc9f1496a8c0808c58142` |
| Deleted template | yes (end of run) |
| Rewind | to sequence 15 (deleted 6 events / 7 memories) |
| Timeline fork | `branch_1781255253389` |
| Session bust | `session:6a2bc9f1496a8c0808c58142` |
| Logs | `/tmp/agent5_batch1.log`, `/tmp/agent5_batch2.log`, `/tmp/agent5_batch3.log` |
