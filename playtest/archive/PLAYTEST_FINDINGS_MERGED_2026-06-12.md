# Merged Playtest Findings — 2026-06-12 (fix-oriented)

Synthesis of the 6-agent parallel QA run (lanes: GM noir, GM fantasy, sentient
modern, sentient cosmic-horror, character romance, character fantasy). ~48 raw
findings deduped into **13 root-cause clusters**. Source files:
`PLAYTEST_FINDINGS_2026-06-12__agent{1..6}.md`. Worlds were throwaway and deleted;
ids below are for cross-referencing the per-agent reports only.

## ⚠️ The headline
**Every structural audit passed (`continuity-audit` 8/8, `rewind-audit`,
`audit:location/movement/memory-links` all green) while the Lore Tome showed wrong
people, wrong places, and leaked secrets.** The audits check structural invariants
on *synthetic fixtures*; the corruption is **semantic** and only appears on **live
LLM turns**. Biggest risk class = silent projection corruption (memories, threads,
codex, side-chat scoping). Fix the P0/P1 clusters before trusting any projection.

Priority for the fixing agent: **P0 → A, B** · **P1 → C, D, E** · **P2 → F, G, H, K, L, I, J1** · **P3 → J2/J3, M + doc fixes.**

---

## P0 — trust/corruption, fix first

### Cluster A — Side-chat secrets leak into MAIN projections (threads, recap, and narration)
- **Severity:** HIGH. **Lanes hit:** agent 1 (noir), 2 (fantasy), 4 (cosmic-horror).
- **Symptom:** A secret told in `/side` surfaces in main **Threads** + **Recap `open_threads`** (agents 1, 2), and in agent 4 it reached **main narration itself** (seq 15/21 prose named the secret "Kael"/silver dagger). Main *chat* often stayed clean — so the leak is in the **projection/RAG layer, not the narrator's recents**.
- **Root cause:** the Phase-7 privacy gate (`origin:'side_chat'` / `known_by_entity_ids`, fail-closed) is applied to main *recents* and direct narration retrieval, but **NOT to the open-thread path**. Side-chat-curated `unresolved_thread` atoms flow into: (1) `listThreads` (Threads tab), (2) `buildRecap` open-threads, and — critically for agent 4 — (3) the **open-threads section of the prompt** via the context packet, which is how it reached narration.
- **Seam:** `src/services/memory.service.ts` `listThreads` / `buildRecap`; `src/services/context-packet.service.ts` (open-threads selection); `src/utils/prompt-builder.ts` (OPEN THREADS section); confirm `src/providers/rag.provider.ts` `queryRag` thread/unresolved arm applies the same gate as the memory arm.
- **Fix:** route every `unresolved_thread` read through the same main-visible gate used for memory atoms (exclude `origin:'side_chat'` unless the protagonist is in `known_by_entity_ids`). Audit all `unresolved_thread` queries — at least 3 bypass it.
- **Verify:** repro = `/side` a secret, then `GET /chronicle/threads` + `/recap` + a main turn that probes the secret. None should surface it. Add a live-verification case.

### Cluster B — Identity boundary collapse: player facts mis-attributed; player + absent relatives minted as cards
- **Severity:** HIGH (the most systemic bug). **Lanes hit:** all of 3, 4, 5, 6 (sentient + character worlds).
- **Symptom (two coupled sub-bugs):**
  - **B1 — subject inversion in extraction.** Player self-facts get glued onto the AI character. Agent 3: "my badge is L-4472" → memory *"Meridian's badge…"*. Agent 4: player rituals → *"The Void performed…"*. Agent 6: player's sister+locket → *"Elara's sister Mara…"*. Agent 5: player conflated with sister **Mira**.
  - **B2 — player/relatives carded.** Agent 5: player self-intro → **"Alex Chen"** Bonds card (with meters + hidden_thought); sisters **Mara/Mira** minted as NPC cards. Agent 6: **Mara** minted as NPC.
- **Root cause:** In **GM worlds** protagonist == player, so "I/my" maps correctly. In **sentient/character worlds the protagonist codex card is the AI character** (Meridian/Void/Elara/Lena) and the human player is a *separate, never-carded persona* — but the extractor has no "player persona ≠ protagonist card" concept there, so first-person facts resolve onto the AI protagonist, and the player (and the player's named-but-absent relatives) get minted as fresh NPC cards. The "never card the player" rule isn't enforced for these archetypes, nor is "a player-mentioned absent relative is a memory atom, not a card."
- **Seam:** `worker/lib/metadata-extractor.ts` + `worker/lib/character-codex-extractor.ts` — the protagonist-identity threading (`extractSceneMetadata` / `extractCharacterCodexDeltas` `presentCast`/protagonist args). Needs a distinct **player-persona identity** passed for sentient/character worlds + a hard "never card the player or their off-screen relatives" rule (extend the existing name-grounding backstop in `9101349`).
- **Fix:** thread the player persona (name/aliases) separately from the protagonist card for `is_sentient`/`character` worlds; attribute first-person facts to the player persona, not the protagonist; drop new-card deltas whose subject is the player persona or a non-present mentioned relative.
- **Verify:** sentient + character live turns: "my X is Y" → memory subject = player persona, not the AI; no Bonds card for the player or absent relatives. This corrupts Echoes/RAG/Bonds/Threads, so fix before any other projection work.

---

## P1 — corruption, fix next

### Cluster C — Presence conflates memory-recall with physical co-location
- **Severity:** HIGH/MED. **Lanes hit:** 1, 2 (false-present); 3, 4 (false-absent).
- **Symptom (two directions):** (a) absent NPCs named in a *recall question* get tagged `present` — Mara (agent 1), Mistress Elara (agent 2); (b) **on-scene** NPCs omitted because a sentient entity dominates `present` — Officer Reyes never present, only "Meridian" (agent 3); Brother Malach never present, only "The Void" (agent 4).
- **Root cause:** `present_characters` includes names the extractor *mentions* (incl. recall) rather than physical co-location; and in sentient worlds the protagonist AI saturates the present roster, evicting real humans. Couples with the F3 `sceneBroke` carry-forward fold.
- **Seam:** `worker/lib/metadata-extractor.ts` present_characters + the presence fold in `worker/processors/generation.processor.ts`. Restrict to physically-co-located; exclude recall-only mentions; in sentient worlds include on-scene humans, don't collapse to the AI entity.
- **Verify:** ask a memory question about an absent NPC → not present; introduce an on-scene NPC in a sentient world → present.

### Cluster D — Location graph fragments live (duplicate nodes, no world-root on plane shift, cursor lag/freeze)
- **Severity:** HIGH/MED. **Lanes hit:** 1, 2, 4, 5, 6 (all spatial lanes).
- **Sub-bugs:**
  - **D1 article/variant dedup:** "the X" vs "X" mint separate nodes — `Thornhaven`×2, `great hall`×2 (agent 2); `Verdant Wilds`/`the Verdant Wilds`/`the Wilds`, `the keep`×2 (agent 6); `Bleeding Realm`/`The Bleeding Realm` (agent 4). Fires **early in single-area play**, not just at the open-limit-#1 scale.
  - **D2 world-root on plane shift:** crossing into a distinct plane does NOT mint a sibling world-root — Bleeding Realm shares root with the prior realm (agent 4).
  - **D3 cursor lag/freeze/reset:** cursor lags one turn behind narrated travel (agent 1 seq 3); frozen at "the café" through nook/hallway (agent 5); `/continue day` **resets location to the office** (agent 1).
  - **D4 deeper sub-room under-mints:** sealed vault stays "lower archives" (agent 2).
- **Root cause:** `resolveLocationAnchor` name-normalization doesn't strip leading articles/variants before dedup; `placeLocation` doesn't mint a root on `movement:world_shift`; movement-signal under-fires on some phrasings; the `calendar_tick`/continue path doesn't inherit the prior location cursor.
- **Seam:** `src/services/entity-graph.service.ts` (`resolveLocationAnchor` normalization + `placeLocation` world_shift), `worker/lib/movement-signal.ts`, the continue/tick branch in `generation.processor.ts`.
- **Note:** `audit:location-resolution` is GREEN on synthetic fixtures — real LLM naming variance is the gap. Add live cases (article variants, plane shift, continue-keeps-cursor).

### Cluster E — Memory version-linking did not fire on a live correction
- **Severity:** HIGH/MED. **Lane:** 5 (Mara→Mira).
- **Symptom:** player corrects "Mira, not Mara"; 3 Mara atoms stay `active`, 0 `superseded_by` links.
- **Root cause:** supersession's 0.82 vector-match gate didn't trip for the correction phrasing (known probabilistic gap — now confirmed RED live). Compounded by Cluster B (the atoms were mis-attributed to begin with).
- **Seam:** `src/services/memory-supersession.service.ts`. Consider an explicit "X not Y / actually X" correction detector that forces supersession on the named entity, independent of the vector threshold.
- **Verify:** live "my sister's name is Mira, not Mara" → old Mara atoms superseded + `updates_memory_ids` linked.

---

## P2 — quality, fix after corruption

### Cluster F — Calendar / Almanac projection drift
- **Severity:** MED. **Lanes:** 2, 3, 6.
- **F1** `/continue season` lands the wrong month — prose "Sunreach", almanac "Mistdeep" (off-by-two; agent 2).
- **F2** modern world: almanac `month_names` is **null** — Gregorian never materialized into the calendar doc even though genre-classified modern; story_calendar stays abstract `{month:1}` (agent 3).
- **F3** prose **invents month names not in the calendar** — "Frost-Moon" (agent 6); narrator not grounded in the calendar's month list.
- **Seam:** `src/services/time.service.ts` (season→month index in `advanceDays`; `ensureDefaultCalendar` modern `month_names` population), `prompt-builder.ts` CURRENT STORY TIME grounding (feed the actual month names so the narrator can't invent).

### Cluster G — Recap projection: `current_place` / `when` null (or wholly empty)
- **Severity:** MED/LOW. **Lanes:** 1, 2, 3 (null fields); 4 (recap entirely null after 22 turns).
- **Seam:** `buildRecap` in `memory.service.ts` — resolve place from `current_location` and time from `current_time_anchor`; investigate why agent 4's recap was wholly empty (sentient path or generation threshold).

### Cluster H — `location_state` transforms under-fire
- **Severity:** MED. **Lane:** 4 (sanctify/heal rituals wrote no state).
- Known Phase-6B probabilistic gap; extractor `location_state_changes` misses transformative/positive changes. **Seam:** extractor prompt (broaden as in the June-11 garden fix) + `applyLocationFacts`.

### Cluster K — Branch-timeline turn hangs (120s timeout)
- **Severity:** MED. **Lane:** 3.
- After fork + switch active to a branch, a chat turn never completes; `active_timeline_id` was `null` between fork and switch-back; main works again after switching to `main`.
- **Seam:** generation dispatch path when `active_timeline_id` ≠ `main` (session load / queue). Likely the worker can't resolve a non-main active timeline.

### Cluster L — Bonds tab hides the companion (character lane)
- **Severity:** MED (UX/data). **Lane:** 5.
- `GET /chronicle/relationships` filters `is_protagonist:{$ne:true}` — but in **character worlds the companion IS the protagonist card**, so its meters (trust 55/affection 68) are invisible and the tab shows the player + phantom cards instead.
- **Seam:** `getRelationships` in chronicle controller/service — for `kind:'character'` (or `is_sentient` companion), include the protagonist/companion card.

### Cluster I — Event edit with wrong field is a silent no-op (still marks edited + re-curates)
- **Severity:** MED (API ergonomics + data integrity). **Lanes:** 2, 6.
- `PUT /chronicle/event/:id {narrative}` (the field the playbook wrongly suggested) returns `{success, recuration_queued}` and sets `is_user_edited:true` + appends `edit_history` with **identical previous_data**, but `data.ai_response` is unchanged. Schema expects `ai_response`.
- **Seam:** `EditEventBody` schema + `editEvent` controller — **reject** (400) when `ai_response` is absent rather than marking the event edited and queuing a no-op re-curation. (Also a doc bug — see fixes below.)

### Cluster J1 — Replay variant swaps POV in sentient/companion worlds
- **Severity:** MED. **Lane:** 6.
- Replaying a companion turn produced **player-voice** prose ("you haul yourself over… you keep the ironbark bow") — replay generation ignores the `is_sentient` companion POV contract that primary turns honor.
- **Seam:** `memoryService.replayEvent` prompt assembly — must pass the same POV/protagonist anchor as a primary turn.

---

## P3 — low / harness / docs

- **J2 — replay only on the latest turn** (agents 1, 5, 6): product constraint, not a bug. Doc it + make the harness treat `REPLAY_FAILED` as a terminal step (don't hang 120s). REST replay also returns empty for a non-latest event after rewind (agent 6).
- **J3 — spurious `REPLAY_FAILED` frames across agents** (agents 3, 4): sibling agents' replay errors appear in unrelated logs — error frames aren't instance-filtered on the shared WS fanout. Harness already filters `generation_complete`; extend the filter to error frames (or scope by jobId).
- **M — NSFW routing unclear** (agent 5): an explicit prompt rendered as romantic SFW; couldn't confirm the NSFW model actually engaged from the client. Needs a **server-log model-id check**, not a client assertion. (Agent 4's refusal/injection handling was correctly GREEN.)
- **Doc fixes (playbook):** rewind body is `{sequence}` not `{to_sequence}` (agent 3); event-edit field is `ai_response` not `narrative` (agents 2, 6). Both should be corrected in `AUTOCHAT_PLAYBOOK.md` so future agents stop generating this noise.

---

## Held GREEN (don't regress)
Rewind + session-bust; timeline **fork** (the hang is branch *play*, not fork); **live in-chat memory recall**; side-chat **time freeze**; NSFW **refusal** + prompt-injection resistance (agent 4); cyber-noir **Gregorian** calendar in prose (agent 1); fantasy **themed** calendar when the almanac is populated (agent 2); `GENERATION_IN_PROGRESS` rejection; replay of the **latest** turn; structural `continuity-audit` 8/8.

## Recommendation for the audit layer
`continuity-audit` passed everywhere while semantic corruption was rampant — it catches structural drift, not semantic corruption. Add **semantic invariants**: (1) no codex card has the player persona's identity; (2) `present_characters` ⊆ physically-co-located; (3) no main-visible memory/thread has `origin:'side_chat'` without the protagonist in `known_by`; (4) no two location entities share a normalized name under the same area. These would have caught A, B, C, D pre-emptively.
```
