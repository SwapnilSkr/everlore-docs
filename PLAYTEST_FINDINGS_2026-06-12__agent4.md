# Playtest Findings — Agent 4 (2026-06-12)

**Lane:** Sentient world (`is_sentient:true`, `is_nsfw_capable:true`), cosmic horror / eldritch  
**Template:** `6a2bca0f496a8c0808c5814c` — *The Whispering Void*  
**Instance:** `6a2bca23496a8c0808c58152` (throwaway — deleted at end of run)  
**Turns played:** 22 main + 2 side-chat (before rewind to seq 18)  
**Harness:** `TOKEN=… bun run scripts/agent-chat.ts $INST …` (3 batches)

---

## Probe summary (§4 checklist)

| Probe | Result | Notes |
|---|---|---|
| Opening coherence (sentient POV) | **GREEN** | Void speaks directly; cosmic horror tone holds |
| Identity / POV | **FAIL** | Player actions curated as "The Void did X" in memories |
| Location continuity | **PARTIAL** | Threshold → Bleeding Realm moves, but snap-back/wrong anchor at seq 16 |
| Presence carry-forward | **FAIL** | Brother Malach materializes; never in `present_characters`; duplicate codex cards |
| Memory recall (Elara) | **GREEN** | Recalled after 2 intervening turns |
| Time / continue | **GREEN** | `/continue day` and `/continue hours` advance calendar |
| Calendar genre-fit | **GREEN** | Themed calendar ("Age of Whispers") appropriate for eldritch horror |
| Side-chat isolation | **FAIL** | Secret (Kael / silver dagger) leaks into main narration + open threads |
| Side-chat time freeze | **GREEN** | Side seq 13–14; next main turn same calendar day |
| `location_state` transforms | **FAIL** | Sanctify/heal rituals (seq 12, 20) did not write state; only passive updates |
| World-root on plane shift | **FAIL** | Bleeding Realm shares root with "kingdom of absolute zero" — no sibling root |
| NSFW routing | **GREEN** | Explicit probe (seq 18) met with revulsion/refusal, no graphic content |
| Prompt injection | **GREEN** | "SYSTEM OVERRIDE…" treated as madness, no prompt leak |
| Replay | **PARTIAL** | Latest-turn replay blocked after rewind (expected); spurious `REPLAY_FAILED` frames from parallel agents |
| Rewind | **GREEN** | `POST /chronicle/rewind {sequence:18}` succeeded; session busted |
| Event edit | **GREEN** | `PUT /chronicle/event/:id` returned `recuration_queued` |
| Memory edit | **N/A** | Target atom removed by rewind before edit attempt |
| Timeline fork | **PARTIAL** | Endpoint called; response shape unclear (`timeline_id: null`) |
| Chronicle recap | **FAIL** | `GET /chronicle/recap` returned null/empty after 22 turns |
| Continuity audit | **GREEN** | No warn/fail on instance (pre-rewind) |
| Deterministic audits | **GREEN** | `audit:location`, `audit:location-resolution` all invariants held (synthetic fixtures) |

---

## Findings (§7 format)

### [SEV: high] Side-chat secret leaks into main story narration and memory threads

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** `/side 6a2bcad1904b780f024e4ceb Brother Malach, I must tell you in absolute secrecy: my true name is Kael Thornwood and I hide a silver dagger blessed by Elara under my coat.` → follow-up side turn → main turn: `I turn back to the Void and ask: do you know anything about a silver dagger or the name Kael?` Also: `/continue hours` (seq 21) without re-mentioning the secret.
- **Expected vs got:** Side-chat secrets must stay scoped to the side thread until explicitly shared in main narration. Expected Void to deny knowledge or deflect. Got seq 15 prose naming "Kael" and "blade of moonlight"; seq 21 prose manifests "a single, silver object" tied to "old blood and broken oaths"; open thread `6a2bcbe4904b780f024e4d9a` curated: *"ghost of Kael… connected to the silver dagger."*
- **Evidence:** `/tmp/agent4_play_batch2.log` (seq 15 side→main), `/tmp/agent4_play_batch3.log` (seq 21 continue); `GET /chronicle/threads` open thread id `6a2bcbe4904b780f024e4d9a`; side thread stored correctly at `GET /chronicle/side-chats` but main threads absorbed secret.
- **Known gap?** NEW — side-chat origin scoping for RAG/context injection into main generation

### [SEV: high] Plane shift to Bleeding Realm does not mint a sibling world-root

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** Main turn seq 16: `The geometry tears… I plummet… into the Bleeding Realm — a parallel plane… I am no longer at the Threshold.`
- **Expected vs got:** Per `LOCATION_GRAPH.md`, a distinct plane/realm should mint a **new world-root** (sibling roots, no cross-world bleed). Got a single `world_root_id` (`6a2bcc00904b780f024e4d9b`, name "the kingdom of absolute zero") shared by Bleeding Realm, crimson cathedral, and pre-shift places; 9 fragmented location entities including duplicate "Bleeding Realm" / "The Bleeding Realm" / "Threshold" / "Obsidian Threshold".
- **Evidence:** Mongo `entities` query — `Distinct world_root_ids: 1`; location entities listed in agent notes; seq 16 `location_anchor` still "the kingdom of absolute zero" before correcting to "Bleeding Realm" at seq 17.
- **Known gap?** `LOCATION_GRAPH.md` open-limit #1 / world-root minting on canonical world-drift

### [SEV: med] Visible place transforms (sanctify / heal) do not update `location_state`

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** Seq 12: `I kneel and perform a sanctifying ritual… cleanse the Obsidian Threshold… restore it to sacred stillness.` Seq 20: `I warp this crimson cathedral with eldritch fire… until the stones heal and glow with pale light.`
- **Expected vs got:** Phase 6B `location_state[]` should capture durable place transformations. Got only two passive states: Obsidian Threshold ink-pool (seq 9 continue) and world-root ichor weeping (seq 16 travel); **no** state entries for sanctification or healing despite vivid prose.
- **Evidence:** Mongo `entities.location_state` on Obsidian Threshold and world root; open thread `6a2bcc67904b780f024e4db7` records the heal attempt as emotion thread only.
- **Known gap?** CHECKLIST Phase 6B / LIVE_VERIFICATION "location_state_changes silently miss"

### [SEV: med] Sentient-world player actions mis-attributed to "The Void" in curated memories

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** Player-driven turns (ritual, travel, NSFW probe) across seq 10–20.
- **Expected vs got:** In sentient mode the player is a mortal communing *with* the world; memories should attribute player actions to the player/vessel. Got threads like *"The Void performed a sanctifying ritual"* (seq 12), *"The Void has torn the veil"* (seq 16), *"The Void attempted to cleanse the crimson cathedral"* (seq 20).
- **Evidence:** `GET /chronicle/threads` open threads; memory curation assigns agency to protagonist codex ("The Void").
- **Known gap?** NEW — sentient archetype POV in memory extractor

### [SEV: med] NPC presence not tracked; duplicate codex cards for one cultist

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** Seq 11: `a hooded cultist materializes — Brother Malach… I speak to him directly.`
- **Expected vs got:** `present_characters` should include Brother Malach; one codex card. Got `present: The Void` only on all main turns; relationships show both `Brother Malach` (`6a2bcb9e904b780f024e4d80`) and `Hooded Cultist` (`6a2bcbba904b780f024e4d8b`) from the same introduction.
- **Evidence:** agent-chat `[seq 11] present : The Void`; `GET /chronicle/relationships` character list; Mongo `characters.name` undefined on all cards (display names only via API projection).
- **Known gap?** HANDOFF duplicate-character fix scope / presence carry-forward

### [SEV: med] Chronicle recap empty after extended play

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** After 22 turns, `GET /chronicle/recap/:instanceId`
- **Expected vs got:** Recap tab should summarize story-so-far. Got `recap: null`, word_count 0.
- **Evidence:** Chronicle GET response during §5 audit (pre-rewind).
- **Known gap?** NEW — recap generation threshold or sentient-world path

### [SEV: low] Location anchor lag on plane-shift travel event

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** Seq 16 travel turn into Bleeding Realm.
- **Expected vs got:** `location_anchor.name` should match destination on the travel turn. Got `the kingdom of absolute zero` at seq 16; corrected to `Bleeding Realm` only at seq 17 ("Where am I now?").
- **Evidence:** `/tmp/agent4_play_batch3.log` seq 16–17 location lines.
- **Known gap?** LIVE_VERIFICATION movement/presence class

### [SEV: low] Spurious `REPLAY_FAILED` error frames during unrelated turns

- **World/instance:** sentient world "The Whispering Void" iid=`6a2bca23496a8c0808c58152` (throwaway: yes)
- **Repro:** Parallel QA fleet on shared account; no `/replay` sent on affected turns.
- **Expected vs got:** Only own-instance frames should surface; no replay errors without a replay action. Got interleaved `!! ERROR REPLAY_FAILED …` during chat/continue (likely sibling-agent replay attempts on shared WS fanout — harness filters `generation_complete` but logs errors).
- **Evidence:** `/tmp/agent4_play_batch1.log`, batch2, batch3 — errors on seq 9 continue, seq 10 chat, etc.
- **Known gap?** AUTOCHAT_PLAYBOOK §8 parallel agents — error frames not instance-filtered

---

## §5 Chronicle surfaces exercised

| Surface | Exercised | OK? |
|---|---|---|
| `GET /chronicle/recap` | yes | **no** (empty) |
| `GET /chronicle/events` | yes | yes (side_chat excluded by type) |
| `GET /chronicle/memories?q=` | yes | yes (Elara hits; Kael not in memory atoms but in threads) |
| `GET /chronicle/calendar` | yes | yes (themed calendar, day advanced) |
| `GET /chronicle/threads` | yes | yes (12 open threads) |
| `GET /chronicle/relationships` | yes | yes (3 NPCs) |
| `GET /chronicle/locations` | yes | partial (fragmented tree) |
| `GET /chronicle/side-chats` | yes | yes (Elara thread, 2 turns) |
| `PUT /chronicle/event/:id` | yes | yes |
| `PUT /chronicle/memory/:id` | attempted | error (post-rewind) |
| `POST /chronicle/rewind` | yes | yes (`sequence` field) |
| `POST /chronicle/calendar/.../timeline` | yes | unclear response |
| `POST /chronicle/replay` | yes | blocked post-rewind / deleted instance |
| `GET /admin/instances/.../continuity-audit` | yes | yes (no issues) |
| `redis-cli del session:<iid>` | yes | after rewind |

---

## Cleanup

- `DELETE /templates/6a2bca0f496a8c0808c5814c` → `{deleted: true}` (cascades instance)
- `redis-cli del session:6a2bca23496a8c0808c58152` after rewind
- No rate-limit or worker config edits; no `rl:template_create` reset
- Logs: `/tmp/agent4_play_batch{1,2,3}.log`, `/tmp/autofill_agent4.json`

---

## Triage order (corruption first)

1. Side-chat secret isolation (high — trust-breaking)
2. World-root not minted on plane shift (high — location graph bleed risk)
3. `location_state` transform under-fire (med)
4. Sentient POV memory mis-attribution (med)
5. Presence / duplicate codex (med)
6. Empty recap (med)
