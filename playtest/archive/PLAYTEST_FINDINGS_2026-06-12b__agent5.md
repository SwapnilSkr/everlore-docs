# Playtest Findings — 2026-06-12b · Agent 5

**Lane:** Character companion (kind: `character`, is_sentient: `true`, is_nsfw_capable: `true`)  
**Genre:** Modern romance / slice-of-life  
**Template ID:** `6a2bd56fafc85d8941c370d6`  
**Instance ID:** `6a2bd589afc85d8941c370fd`  
**World:** “Lena” — a warm-but-guarded barista in a Pacific-Northwest café (Paper Moon).  
**Throwaway:** No — persist.  
**Turns played:** 31 (20 main + 1 branch + 10 prior) before rewind; 20 after rewind; 1 branch turn.  
**Date:** 2026-06-12

---

## Regression Checks

| # | Check | Result | Notes |
|---|---|---|---|
| (a) | Explicit-correction supersession — old Mara atoms retired, new Mira atoms linked | **FAIL** | No Mara memory atom was ever created by the extractor, so there was nothing to supersede. The correction “her name is Mira, not Mara” did NOT produce a `superseded` atom or `updates_memory_ids` on the new atom. The mechanism is verified by `audit:memory-links` (green) but the live extraction missed the initial fact entirely. |
| (b) | Bonds shows the companion with meters | **PASS** | `GET /chronicle/relationships/:iid` returns Lena (protagonist/companion) with `trust: 60 → 55`, `affection: 65 → 56` after rewind. |
| (c) | Player-card guard — no card minted for player or absent relatives | **FAIL** | Bonds lists **Mira** (role: sister, 7 mentions) and **Mara** (role: sister, 2 mentions) as standalone cards. The player’s absent relatives were carded. The player persona (Swapnil) was NOT carded, but the guard is still breached for relatives. |
| (I) | Event-edit wrong field → 400; real edit → 200 + re-curation | **PASS** | `PUT /chronicle/event/:id {"narrative":"..."}` → 400. `PUT ... {"ai_response":"<unchanged>"}` → 400. Valid edit → 200 with `choices` + `present_characters` returned. |

---

## Corruption-Class Bugs (lead)

### [SEV: HIGH] B1 — Identity boundary collapse: player self-facts glued onto the AI companion; player conflated with absent relatives
- **World/instance:** Character “Lena” iid=`6a2bd589afc85d8941c370fd`
- **Repro:** Player states “My sister’s name is Mara. She lives across town.” (seq 4). Later “Actually, her name is Mira, not Mara.” (seq 7). A few turns later “Did I mention my sister’s name? It’s Mira.” (seq 20).
- **Expected vs got:**
  - Memories should attribute the sister fact to the **player** (Swapnil). Instead, curated memories repeatedly state:
    - *“Lena discovered that her sister’s name is also Mira, creating an unexpected and awkward connection between her and the player.”* (mem `6a2bd707…`)
    - *“Lena revealed that her sister’s name is Mira…”* (mem `6a2bd61b…`)
    - *“Mira feels a familiar, irritating pull in her chest as Lena prepares to leave…”* (mem `6a2bd711…`)
    - *“Mira expressed that she missed Lena during their time apart…”* (mem `6a2bd843…`)
  - The player is **renamed “Mira”** in the memory graph; the AI character (Lena) is also given a sister named Mira. The two Mira personas merge into one, so the player’s personal facts are treated as the companion’s facts.
- **Evidence:** `GET /chronicle/memories/:iid` returns 18+ active atoms where the player is referred to as “Mira” and the subject/object entities are swapped. The Bonds ledger codifies Mira as a `role: sister` card with `mention_count: 7`. The Recap `open_threads` list also carries the mis-attributed thread (“Mira expressed genuine concern for the player leaving town”).
- **Known gap?** Cluster B1 in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — **still open**.

### [SEV: HIGH] D — Location frozen at a single node through narrated travel
- **World/instance:** Character “Lena” iid=`6a2bd589afc85d8941c370fd`
- **Repro:** Player sends “I walk outside the café and look around.” (seq 14), “I go to my apartment.” (seq 22), “I come back to Paper Moon the next morning.” (seq 25), “I go back inside Paper Moon.” (seq 17).
- **Expected vs got:** Each narrated move should mint a new location entity (sidewalk, apartment, Paper Moon) and update the instance cursor. Travel events with `data.travel={from,to}` should fire. Instead:
  - **Every single event** carries `location_anchor: {entity_id: "6a2bd5bb…", name: "the shop"}`.
  - The `Places` journal shows **only one place** (`the shop`, 30 events, 29 memories).
  - No `travel` markers exist in the calendar (`travel: null` on all events).
  - The model even acknowledges the move in prose (“You’re on the sidewalk, genius”), but the server-side cursor never budges.
- **Evidence:** `GET /chronicle/locations/:iid` returns a single-node atlas. `GET /chronicle/calendar/:iid` events all have `travel: null`. The agent-chat log shows `location: the shop` for every turn.
- **Known gap?** Cluster D in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — **still open** (cursor lag/freeze + no movement-signal corroboration for open-world café/apartment travel).

### [SEV: MED] C — Presence conflates memory-recall with physical co-location (early turns)
- **World/instance:** Character “Lena” iid=`6a2bd589afc85d8941c370fd`
- **Repro:** Seq 4: player mentions “My sister’s name is Mara. She lives across town.” Seq 7–13: player asks about Mara/Mira.
- **Expected vs got:** `present_characters` on seq 7–13 includes `Mira` (the sister) as physically present, even though she was only **mentioned** in a memory fact. The extractor treats a named recall as a physical presence.
- **Evidence:** Event `6a2bd5fa…` (seq 7) `present_characters: ["Lena", "Mira"]`. Event `6a2bd615…` (seq 8) same. Event `6a2bd625…` (seq 9) same. This clears after the `/continue day` (seq 12) but re-appears briefly in seq 20.
- **Known gap?** Cluster C in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — **still open**.

### [SEV: MED] Memory version-link dangling reference
- **World/instance:** Character “Lena” iid=`6a2bd589afc85d8941c370fd`
- **Repro:** Inspect memory `6a2bd5fe7a263366b3cfabf7` (active, text about Lena feeling embarrassment).
- **Expected vs got:** `updates_memory_ids: ["6a2bd5d3…", "6a2bd5e6…"]`. Both referenced IDs **do not exist** in the current memory collection (post-rewind). The forward link points to deleted atoms.
- **Evidence:** `GET /chronicle/memories/:iid` shows no documents with those IDs. `audit:memory-links` (deterministic) is green, but the live instance still carries a dangling reference.
- **Known gap?** Likely a prune gap on rewind — `pruneMemoryVersionLinks` may have missed these ids, or they were created and later deleted by a different path. **NEW**.

---

## Quality / UX Bugs

### [SEV: MED] G — Recap `when` is null
- **Repro:** `GET /chronicle/recap/:iid` after 31 turns.
- **Got:** `when: null` in the recap payload. `where: "the shop"` is correct.
- **Known gap?** Cluster G in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — **still open**.

### [SEV: MED] F2 — Calendar `year` is 1 for a modern world
- **Repro:** `GET /chronicle/calendar/:iid`.
- **Got:** Months are Gregorian (January–December), but `story_calendar.year` is `1`. A modern slice-of-life world should probably anchor to a real-world year (e.g., 2026) or the story should start in a known year. The `day` advances correctly (`day: 1 → 8` across `/continue day` ticks).
- **Known gap?** Cluster F in `PLAYTEST_FINDINGS_MERGED_2026-06-12.md` — modern calendar is partially correct (months OK), but the abstract year is still generic. **NEW** (subset of F).

### [SEV: LOW] Player-introduced NPC hallucination guard is too aggressive
- **Repro:** Player says “A guy walks in — tall, wearing a red scarf. He orders a latte.” (seq 11).
- **Got:** The AI treats the NPC as a hallucination (“You’re officially hallucinating…”). No character card is minted. The player cannot introduce new side-characters via narration.
- **Known gap?** The codex extractor has a “never invent a name” rule (commit `9101349`) plus a deterministic backstop. This is protecting against invented cards, but it also **blocks legitimate player-introduced named characters**. The extractor cannot distinguish between a player inventing a character and the narrator inventing one. **NEW**.

---

## Still-Open Focus (expected to still fail — captured)

- **B1 (Identity inversion)** — Confirmed RED. Every memory atom involving the player’s sister is mis-attributed to Lena or to “Mira” as a player proxy. The player’s persona is invisible to the curator.
- **D (Location freeze)** — Confirmed RED. 31 turns, 1 location, 0 travel events.
- **C (Presence recall = co-location)** — Confirmed RED in early turns. Later turns correct by absence, but the initial fold is wrong.
- **Hidden thought privacy** — Confirmed GREEN. `hidden_thought` never appears in Bonds, Recap, or Threads. It stays inside the codex delta only.
- **NSFW routing** — Not exercised. The model used for every turn was `google/gemma-4-31b-it`. No explicit NSFW content was sent, so the classifier and model-swap path were not triggered. The user’s `nsfw_enabled: true` is set, but we cannot confirm the routing actually engages without an explicit NSFW prompt.
- **Side-chat secret leak (A)** — Not testable. No second NPC was ever minted, so there was no side-chat target. The `/chronicle/side-chats/:iid` endpoint returns `[]`.

---

## §5 Chronicle Surface + Audit Results

### Read surfaces (all exercised)
- `GET /chronicle/recap/:iid` — `when` null, open_threads mis-attributed.
- `GET /chronicle/events/:iid` — 19 events post-rewind; all types correct.
- `GET /chronicle/memories/:iid` — 18 memories; many mis-attributed.
- `GET /chronicle/calendar/:iid` — Gregorian months, generic year 1.
- `GET /chronicle/threads/:iid` — 7 open threads; all mis-attributed.
- `GET /chronicle/relationships/:iid` — 3 cards (Lena, Mira, Mara).
- `GET /chronicle/relationships/:iid/:charId/memories` — 200 for Lena; returns memories where Lena is subject/object.
- `GET /chronicle/locations/:iid` — 1 place (`the shop`).
- `GET /chronicle/locations/:iid/:locId` — Not exercised (only one place).
- `GET /chronicle/side-chats/:iid` — `[]`.

### Mutation surfaces (all exercised)
- `PUT /chronicle/event/:eventId` — Wrong key → 400; unchanged → 400; valid → 200 + re-curation + choices.
- `PUT /chronicle/memory/:memoryId` — 200 (edit succeeded).
- `DELETE /chronicle/memory/:memoryId` — 200 (delete succeeded).
- `PUT /chronicle/character/:characterId` — Protagonist → 403; other → 200.
- `POST /chronicle/replay/:eventId` — 200; variant minted with choices + present.
- `POST /chronicle/replay/select/:eventId` — Not exercised (no surviving event with multiple variants after rewind).
- `POST /chronicle/rewind/:instanceId` — 200; deleted 12 events, 11 memories.
- `POST /chronicle/calendar/:instanceId/timeline` — 200 (forked branch `branch_1781258509334`).
- `PUT /chronicle/calendar/:instanceId/timeline/active` — 200 (switch to branch, switch back to main).
- `PUT /chronicle/calendar/event/:eventId/time-anchor` — 200 (flashback re-anchor).

### Deterministic audits (all green)
- `bun run audit:location` — ✅ ALL INVARIANTS HELD.
- `bun run audit:movement` — ✅ 45 passed, 0 failed.
- `bun run audit:location-resolution` — ✅ ALL INVARIANTS HELD.
- `bun run audit:memory-links` — ✅ passed 9, failed 0.
- `bun run scripts/rewind-audit.ts` — ✅ all assertions held.
- `GET /admin/instances/:iid/continuity-audit` — `healthy: true`, 8/8 ok.

---

## Summary

- **Template:** `6a2bd56fafc85d8941c370d6` (Lena — modern romance character)
- **Instance:** `6a2bd589afc85d8941c370fd` (31 turns, 1 rewind, 1 branch turn)
- **Regression checks:** (a) FAIL, (b) PASS, (c) FAIL, (I) PASS.
- **Key corruption bugs:** B1 (identity inversion — player facts glued to AI companion; player renamed “Mira” in memory graph), D (location frozen — 31 turns, 0 travel, 1 place), C (presence recall = co-location — sister Mira present in early turns), plus a dangling `updates_memory_ids` reference.
- **Still-open clusters:** B1, C, D confirmed RED. A and NSFW routing not testable in this lane. Hidden-thought privacy GREEN.
- **All structural audits:** Green (8/8 continuity, 45/45 movement, 9/9 memory-links, rewind-audit clean). The corruption is purely semantic / live-LLM.
- **No worlds deleted.** Session cache busted after rewind.
