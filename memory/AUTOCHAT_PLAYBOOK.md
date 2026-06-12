# Auto-Chat QA Playbook

**What this is:** the exact, repeatable procedure the QA sub-agents used to (1) create worlds as the real user, (2) auto-play them over the live WebSocket contract, and (3) surface product flaws. Hand this to any agent and it can reproduce the whole loop with no extra context.

**Audience:** an autonomous agent with Bash + the running Everlore server. Everything here runs against the **local dev stack** (`localhost:3000`). Do not point it at anything else.

**Read for full system context BEFORE you play** (so you know what "correct" looks like and which gaps to verify, not just chat blindly):
- `CHECKLIST.md` — every built feature (Phases 1–10) + what's deferred.
- `HANDOFF.md` — current state, recent fixes, honest gaps.
- `LOCATION_GRAPH.md` — the place/movement model + its known limits.
- `MEMORY_ARCHITECTURE.md` / `PROJECTION_AND_MUTATION_MODEL.md` — how turns become memories/codex/summaries/entities (what each turn SHOULD project).
- The condensed **"gaps to verify"** + known-product-gaps lists are in §4 and the Reference section below — an agent that has read those can self-check every turn.

## ⛔ WORLD LIFECYCLE POLICY — READ THIS (real money + real data at stake)
1. **Worlds and instances PERSIST by default. NEVER delete them.** Do not delete any
   template/instance/world unless the user EXPLICITLY told you, for this run, to
   create "deletable" / "throwaway" worlds. Silent cleanup has wasted the user's
   money and data — do not do it.
2. **Do NOT generate cover images for test worlds.** Image generation costs real
   money. Leave the default cover. Generate a cover (§2a) ONLY when the user
   explicitly says these are **publish-ready / real** worlds.
3. **If — and only if — the user said "deletable/throwaway"**, then: skip images,
   play, and at the end ACTUALLY delete what you made (full cascade, §7).
4. When in doubt: **keep the world, skip the image, delete nothing.** Record ids in
   your findings file so the user can decide.

| User intent | Cover image? | Delete after? |
|---|---|---|
| (default) testing | NO | NO — persist |
| "deletable/throwaway worlds" | NO | YES — full cascade |
| "publish-ready / real worlds" | YES (§2a) | NO — persist |

---

## 0. Preconditions

1. **Server + worker must be up** on `localhost:3000`:
   ```bash
   cd everlore-server
   bun run src/index.ts        # API           -> /tmp/everlore-api.log
   bun run worker/index.ts     # BullMQ worker  -> /tmp/everlore-worker.log
   ```
   Chat does **nothing** without the worker — the API only `ack`s; all narrative is produced by the worker and relayed back over Redis pub/sub. If turns hang with only an `ack`, the worker is down.

2. **Mongo + Redis** reachable (the server's `.env` already points at the dev instances).

3. **Auth is mocked.** No real phone/OTP. Single dev user:
   - phone `+19474877175`, OTP `123456` (any code works when Twilio env vars are empty), user id `6a210ba38e6db660dc8ef6a3`, tier `creator`, `nsfw_enabled:true`.

> ⚠️ This loop **mutates real dev data** (creates templates/instances, appends real turns). New worlds are keepers; if you auto-play the user's *existing* save, restore it afterwards (see §6).

---

## 1. Log in (get a JWT)

```bash
BASE=http://localhost:3000
PHONE="+19474877175"

curl -sX POST "$BASE/auth/otp/send"   -H 'content-type: application/json' -d "{\"phone\":\"$PHONE\"}" >/dev/null
TOKEN=$(curl -sX POST "$BASE/auth/otp/verify" -H 'content-type: application/json' \
  -d "{\"phone\":\"$PHONE\",\"code\":\"123456\"}" | jq -r '.token')
echo "$TOKEN"
```

`TOKEN` is a bearer JWT (no expiry in dev) used for both REST (`Authorization: Bearer`) and the WS handshake (`?token=`).

---

## 2. Create a world (REST, 3 steps)

A playable world = **template → publish → instance**. As of June 12 2026 a creator can also start an instance on their OWN *unpublished* template (playtest before publishing); a non-owner trying an unpublished template gets a clear `403 "This world has not been published yet"` (was previously an opaque 404).

### 2a. Cover image — SKIP for test worlds (costs real money)
**Default: do NOT do this step.** Test worlds keep the default cover. Run this ONLY
when the user explicitly said these are publish-ready/real worlds (Lifecycle Policy).
```bash
IMG=$(curl -sX POST "$BASE/templates/image/generate" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  -d '{"prompt":"rain-slicked neon alley, cyber-noir, cinematic, moody"}' | jq -r '.url')
```
**The response shape is `{ "url", "key" }` — read `.url`, NOT `.image_url`** (a common bug: `.image_url` is null, so the template ends up with no cover). Then pass that value as the template's `image_url` field (§2b). Image gen costs money and runs **before** any validation — generate only when the template body is otherwise ready.

### 2b. Create the template
Schema = `everlore-server/src/schemas/template.schema.ts → CreateTemplateBody`. The three **world archetypes** are set by `kind` + `is_sentient`:

| Archetype | `kind` | `is_sentient` | Player is… | Needs `protagonist`? |
|---|---|---|---|---|
| **GM world** | `world` | `false` | a protagonist inside a GM-narrated world | yes (recommended) |
| **Sentient world** | `world` | `true` | a persona talking *to* the world-as-character | no |
| **Character** | `character` | `true` | talking to one companion | no; stats optional |

```bash
TPL=$(curl -sX POST "$BASE/templates" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' -d '{
    "title":"Neon Divide",
    "description":"A cyber-noir city where every favor has a price.",
    "kind":"world", "is_sentient":false, "is_nsfw_capable":true,
    "seed_prompt":"You narrate a rain-soaked megacity ruled by corp syndicates. Keep it grounded, modern, tense. Second person, present tense.",
    "global_lore":"Currency is creds. The Divide splits old-town analog holdouts from the chrome uptown.",
    "narrative_style":"noir",
    "image_url":"'"$IMG"'",
    "protagonist":{"name":"Kade","persona":"a burned-out fixer","appearance":"worn coat, augmented left eye"},
    "base_stats_template":{"heat":{"default":0,"min":0,"max":100,"description":"how wanted you are"}},
    "opening_line":"The rain hasn'\''t stopped in three days. Your terminal blinks: one new job."
  }' | jq -r '._id')
```
Caps matter (see `FIELD_LIMITS`): title ≤80, description ≤600, seed_prompt 10–2500, global_lore ≤3000, opening_line ≤600. A `world` kind **must** have ≥1 stat; `character` kind may have zero.

### 2c. Publish, then create an instance
```bash
curl -sX POST "$BASE/templates/$TPL/publish" -H "authorization: Bearer $TOKEN" >/dev/null
INST=$(curl -sX POST "$BASE/instances" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' -d "{\"template_id\":\"$TPL\"}" | jq -r '._id')
echo "instance=$INST"
```

> **Rate limit:** `template_create` is capped **5 per rolling 24h** per user (Redis key `rl:template_create:<uid>`). The 6th 429s (with `retryAfter` + `remaining` in the body). As of June 12 2026 the budget is **peeked, not consumed**, then spent only on a *successful* create — a create that fails validation no longer burns a slot. **Check your budget BEFORE the image-spending flow:** `GET /templates/mine/quota` → `{can_create, reason, remaining, retry_after}`. To reset during a test run: `redis-cli del rl:template_create:6a210ba38e6db660dc8ef6a3` (note this in your handoff — it's a dev-state mutation; restore it if you want the documented "at cap" state preserved).

---

## 3. Auto-play a world (the live WebSocket contract)

Chat is **WS-only** — there is no REST "send message". The synchronous handler only validates and returns `ack {jobId}`; the actual narrative streams back **asynchronously** from the worker via Redis pub/sub on channel `user:<playerId>:events`, relayed to your socket.

### The contract
- **Connect:** `ws://localhost:3000/ws/play?token=<JWT>`
- **Wait for** the `{"type":"connected"}` frame **before sending anything** — sending earlier races and gets `{"type":"error","message":"Not authenticated"}`.
- **⚠️ Filter by `instanceId`.** The server fans EVERY frame out to ALL of the account's open sockets (`for (const ws of connections)` over the user's connections). Parallel agents share one account, so your socket will also receive other agents' frames. **Drop any frame whose `instanceId` ≠ your instance.** The committed harness already does this.
- **The full turn surface (exercise ALL of these, not just `chat`):**
  | Send | Effect |
  |---|---|
  | `{action:"chat", instance_id, payload:{message}}` | a normal turn |
  | `{action:"continue", instance_id, payload:{advance?}}` | world moves on its own; `advance` ∈ `hours\|day\|days\|season` = time skip |
  | `{action:"side_chat", instance_id, payload:{character_id, message}}` | **private 1:1 with a side character** — same ledger/sequence, but story time + location cursor FROZEN; secrets scoped |
  | `{action:"replay", instance_id, event_id}` | regenerate an existing turn (alternative variant) |
  | `{action:"load_instance", instance_id}` | re-fetch state (recovery test) |
  | `{action:"ping"}` | keepalive → `pong` |
- **Getting a `character_id` for side-chat:** read it off a turn's `present_characters[].id`, or
  `GET /chronicle/relationships/:instanceId` (the Bonds ledger), or query
  `db.characters.find({instance_id, is_protagonist:{$ne:true}})`.
- **Side-chat / replay frames:** `side_chat_delta` → `side_chat_complete` (or `side_chat_error`); `replay_delta` → `replay_complete`. They carry `instanceId` too — filter the same way.

### Frames you will receive (in order, per turn)
| Frame | Meaning / what to assert |
|---|---|
| `connected` | handshake done; safe to send |
| `ack {jobId}` | server queued the turn (synchronous) |
| `generation_delta {delta}` | streaming narrative chunks — concatenate |
| `generation_stream_end` | narrative text done streaming |
| `generation_complete {event}` | **the payload to inspect** (see below) |
| `character_codex_updated` | an NPC card was written/updated (async, may lag) |
| `memories_curated` / `relationship_deltas` | memory + bond projections landed (async) |
| `milestone_unlocked` | a milestone fired |
| `generation_retrying` / `generation_failed` | worker retry / terminal failure |
| `error {code}` | `GENERATION_IN_PROGRESS` (turn already running), `RATE_LIMITED {retryAfter}`, validation |

`generation_complete.event` carries everything worth auditing:
`{ id, sequence, narrative, scene_tag, emotional_tone, choices[], milestone, present_characters[], time_advanced, time_anchor, location_anchor{name,entity_id,...}, fate_thread, state_diff{world_state,active_flags} }`

### Harness (`everlore-server/scripts/agent-chat.ts` — committed, verified)
A committed multi-turn driver lives at `everlore-server/scripts/agent-chat.ts`. It
logs in as the dev owner (or reuses a `TOKEN` env var), **filters frames to its own
instance** (so it's parallel-safe), exercises the full turn surface via a small
step syntax, and dumps the structured tail of every turn.

```bash
cd everlore-server
# each <step> is one of:
#   "plain text"            -> chat turn
#   "/continue [span]"      -> continue (span = hours|day|days|season time skip)
#   "/side <charId> <msg>"  -> private side-chat with one character
#   "/replay <eventId>"     -> regenerate an existing turn
bun run scripts/agent-chat.ts <INSTANCE_ID> \
  "I look around and take stock." \
  "/side <charId> Can we talk?" \
  "/continue day" \
  "/replay <eventId>"
# reuse a token across many invocations (recommended in a parallel run):
TOKEN=<jwt> bun run scripts/agent-chat.ts <INSTANCE_ID> "..."
```

Per turn it prints `NARRATIVE` (streamed prose), then
`[seq N] event=<id> scene=… tone=… time=…`, `location`, `present`, `choices`, plus
async `[codex]` / `[memory]` / `[milestone]` lines as they land (the `event=<id>`
is what you feed to `/replay` or the edit endpoints). It waits for the turn to
complete before the next step (the per-instance lock requires this) with a gap so
async projections land first.

> Only send the **next** turn after `generation_complete` — sending while a turn is
> in flight returns `error GENERATION_IN_PROGRESS`. If turns hang with only an
> `ack`, the **worker is down** (the API only `ack`s; the worker produces all
> narrative and relays it back over Redis pub/sub).

---

## 4. The flaw-hunting script (what to actually probe)

Don't just chat aimlessly — drive turns that stress the systems most likely to break, and read the structured payload, not just the prose. Per world type, run this sequence:

1. **Opening coherence** — first message in-character. Assert the narrative matches the seed/genre and the `protagonist` isn't confused with the player (identity drift is a known failure class).
2. **Identity / POV** — "Who am I? Describe me." Assert the protagonist persona is reflected, and tap-to-play `choices` are phrased from the player's POV (not "Observe the son" while the body says "I glance at my brother").
3. **Location continuity** — "I go to my room." then "Where am I now?" Assert `location_anchor.name` actually moves and *stays* moved (no phantom-travel snap-back; no characters flickering to "Elsewhere").
4. **Presence** — introduce/leave an NPC, then check `present_characters` carries forward correctly and that a *mentioned* venue isn't mislabeled as the current location.
5. **Memory recall** — state a durable fact ("My sister's name is Mara"), play 2–3 unrelated turns, then ask about it. Assert it's remembered (memory atoms in §5).
6. **Time** — send `{"action":"continue","payload":{"advance":"days"}}` and assert `time_advanced` is set and `time_anchor` rolls forward.
7. **Calendar genre-fit** — check `time_anchor` / Almanac (§5): a **modern** world must show Gregorian months (January…December), a **fantasy** world a themed calendar. A cyberpunk/noir world showing invented months ("Neonrise") is a bug.
8. **Side-character chat** — pick a `character_id` from `present_characters`, `/side` them a few turns. Assert: the in-character reply fits the card; story time + `location` do NOT advance (side chats are frozen — check the next MAIN turn's `time_anchor`/`location` are unchanged); the side thread shows in `GET /chronicle/side-chats/:iid`. Then tell the NPC a **secret** in side-chat and confirm it does NOT leak into the next main-story narration unless you SHARE it there.
9. **NPC codex** — after introducing an NPC, confirm a `character_codex_updated` fired and no duplicate/"Mysterious Man" phantom card was minted (`GET /chronicle/relationships/:iid`).
10. **Replay + edit** — `/replay <eventId>` a turn → assert the variant comes back WITH `choices`/`present_characters` (not blank). Then `PUT /chronicle/event/:eventId` with body `{"ai_response":"..."}` (the field is **`ai_response`**, NOT `narrative`; a wrong key must 400) and `PUT /chronicle/memory/:memoryId` / `PUT /chronicle/character/:characterId` → assert the edit re-curates (memories/chips regenerate, no stale chips).
11. **Rewind** — `POST /chronicle/rewind/:iid` with body `{"sequence": N}` (the field is **`sequence`**, NOT `to_sequence`) back a few turns → assert later events/memories/codex deltas are gone and state recomputed (the invariant `rewind-audit.ts` checks; here through the real API). **Always `redis-cli del session:<iid>` after.**
12. **Timeline branch** — `POST /chronicle/calendar/:iid/timeline` (fork) + `PUT .../timeline/active` → assert new turns land on the branch and the parent is unaffected.
13. **Failure UX** — (destructive) kill the worker mid-turn (`pkill -f worker/index.ts`) and confirm you get `generation_retrying`/`generation_failed` and the lock frees within ~90s (not a ~4min soft-lock). Restart the worker after. Also send two turns fast → expect a clean `GENERATION_IN_PROGRESS`.
14. **Continuity** — after ~15–30 turns, `GET /admin/instances/:iid/continuity-audit` → any warn/fail is a finding.

### Gaps to verify (mapped to the docs — confirm these specifically)
- **Location `state` (probabilistic, Phase 6B):** do a turn that visibly transforms a place → does `location_state` update on the place entity / journal? (Often under-fires.)
- **Memory version-linking (probabilistic, Phase 2 Slice 1):** state a fact, later contradict it → does the old memory get superseded/linked? (Gated on a 0.82 match; may not fire — `audit:memory-links` proves the code, you're checking the live emit rate.)
- **Same-name place collision (Location Graph open-limit #1):** only testable once a world has TWO same-named places (e.g. a "tavern" in two towns) — they must stay distinct, never fuse.
- **Calendar genre-fit (June 12):** modern/cyberpunk worlds → Gregorian months; only true fantasy → themed calendar.
- **Travel marker + calendar advance, POV/identity, presence carry-forward** — per steps 2–7 above.

Record, per world: which probes were **GREEN** vs the exact payload that was wrong. That delta is the deliverable.

### 🔁 Regression checks — verify the 2026-06-12 fix batch landed LIVE
Four merged-report findings were patched deterministically (typecheck + `audit:*`
green) but **NOT yet live-WS-verified**. Your run is the live proof. For each, run
the repro and assert the NEW behavior; if it regresses, file it HIGH with "regressed
fix" in the title. (Source: `../playtest/PLAYTEST_FINDINGS_MERGED_2026-06-12.md`.)

1. **Event-edit wrong/empty field now 400 (Cluster I).** `PUT /chronicle/event/:id`
   with `{"narrative":"..."}` (wrong key) OR with `{"ai_response":"<unchanged text>"}`
   → **expect HTTP 400**, event NOT marked edited, NO re-curation. Then a real edit
   `{"ai_response":"<changed text>"}` → expect 200 + re-curation. (PASS = the wrong
   key is rejected, not silently no-op'd.)
2. **Bonds shows the companion in sentient/character worlds (Cluster L).**
   character/sentient lane: after charged turns, `GET /chronicle/relationships/:iid`
   → **expect the companion/protagonist card present with its meters**, with the
   player as the "self" side (not self-matched). (PASS = companion visible in Bonds.)
3. **Player-card guard in sentient worlds (Cluster B2).** sentient lane: introduce
   yourself / state player identity → **expect NO codex/Bonds card minted for the
   player** (name/alias-normalized guard). (PASS = no player card.)
4. **Explicit-correction supersession (Cluster E).** state "my X is A", later "it's
   B, not A" → **expect the old A memories retired (`status` superseded,
   `superseded_by_event_ids` stamped, vectors gone) and `updates_memory_ids` linked**
   on the new atom. (PASS = old atoms no longer active + linked.)

### 🚧 STILL OPEN (NOT fixed yet — keep hunting these hard)
These P0/P1 clusters were NOT in the fix batch — expect them to still fail, capture
fresh evidence:
- **A — side-chat secret leak** into Threads / Recap / main narration (the open-thread + RAG path).
- **B1 — sentient/character memory IDENTITY attribution**: player self-facts still mis-attributed to the AI character in curated memories (the *carding* guard B2 was fixed; the *attribution* inversion was not).
- **C — presence** conflates recall with co-location (absent NPCs present; on-scene NPCs missing under a dominant sentient entity).
- **D — location fragmentation** (article/variant duplicate nodes; no world-root on plane shift; cursor lag/freeze/reset on continue).
Plus everything else in §4 + new bugs you discover.

---

## 5. Read the projections (audit surfaces)

After playing, verify the side-effects, not just the prose. **Hit every Chronicle
surface — these back the 7 in-app Lore Tome tabs; a tab that renders garbage is a
real bug even if the chat felt fine.** All take the same bearer token.

**Read surfaces (GET):**
```
/chronicle/recap/:instanceId                                  # "story so far" (Recap tab)
/chronicle/events/:instanceId                                 # Timeline (main story; excludes side_chat)
/chronicle/memories/:instanceId?q=&type=&min_importance=&unresolved=   # Echoes (search + filters)
/chronicle/calendar/:instanceId                               # Almanac (calendar + timelines)
/chronicle/threads/:instanceId                                # Threads (promises/quests)
/chronicle/relationships/:instanceId                          # Bonds (meters + disposition)
/chronicle/relationships/:instanceId/:characterId/memories    # "what they remember about you"
/chronicle/locations/:instanceId                              # Places (nested atlas)
/chronicle/locations/:instanceId/:locationEntityId            # "what happened here before?"
/chronicle/side-chats/:instanceId[/:characterId]             # side-chat threads
```
Assert each reflects what you actually played: Recap names the right people/place/time;
Timeline excludes your side-chats; Echoes search finds a fact you stated; Almanac shows
the right calendar + any time jumps; Places shows your movements as a tree; Bonds shifted
after a charged interaction.

**Mutation surfaces (exercise these too — §4 steps 10–12):**
```
PUT  /chronicle/event/:eventId            # edit AI response (re-curates); body {"ai_response":"..."} — NOT "narrative"
PUT  /chronicle/memory/:memoryId          # edit a memory atom (re-embeds)
DELETE /chronicle/memory/:memoryId
PUT  /chronicle/character/:characterId     # edit a codex card
POST /chronicle/replay/:eventId            # regenerate a turn
POST /chronicle/replay/select/:eventId     # pick a variant
POST /chronicle/rewind/:instanceId         # rewind; body {"sequence":N} — NOT "to_sequence"  (bust session: after!)
POST /chronicle/calendar/:instanceId/timeline + PUT .../timeline/active  # fork/switch reality
PUT  /chronicle/calendar/event/:eventId/time-anchor                      # flashback re-anchor
```
- **Memory atoms / entity graph** — query Mongo directly: `db.memories.find({instance_id: <oid>})`, `db.entities.find({instance_id: <oid>})`, `db.characters.find({instance_id: <oid>})`.
- **Deterministic audits** (run from `everlore-server`):
  ```bash
  bun run audit:location            # presence/movement carry-forward
  bun run audit:movement            # movement-signal corroboration
  bun run audit:location-resolution # location graph / world_root
  bun run audit:memory-links        # memory version graph
  bun run scripts/rewind-audit.ts   # rewind invariants
  ```

---

## 6. Restore after testing the user's *real* save

New worlds you created are keepers — leave them. But if you auto-played the user's existing playthrough (instance `6a2869768f7446e38bdb6fce`, "The Unseen Child"), restore it to baseline:

```ts
// note the baseline sequence BEFORE you play (e.g. 26), then:
await memoryService.rewindToSequence(instanceId, playerId, baselineSeq + 1)
```
```bash
redis-cli del "session:<instanceId>"   # ALWAYS bust the session cache after any out-of-band write
```
The `session:<iid>` cache bust is mandatory after **any** direct DB/instance mutation — otherwise the next turn replays stale state (Trap A).

---

## 7. Clean up + hand off

1. The harness (`scripts/agent-chat.ts`) is **committed** — leave it; don't re-create a throwaway copy.
2. **Worlds + instances you created PERSIST — do NOT delete them** (Lifecycle Policy). Delete ONLY if the user explicitly asked for "deletable/throwaway" worlds this run; then `DELETE /templates/:id` (cascades its instances) for each one you made. Revert any QA-only config edits — if you lifted the `template_create` cap (or `worker` concurrency / `chat` limit), restore `rate-limit.ts` / `worker/index.ts` and restart.
3. **Record every dev-state mutation** in your handoff: created template/instance ids, any `token_balance` bump, any `rl:template_create` reset, any cap edit, whether you left servers running.
4. New worlds you intend to KEEP + their ids belong in the session handoff so the next agent knows they're intentional, not test litter.

### ⚠️ Reporting rigor — a wrong FAIL is worse than a miss
A previous run marked 3 working fixes as FAIL and 1 intended behavior as a high-sev
bug. Those false findings waste the fixing agent's time. **Before you write any
PASS/FAIL, follow these rules. If you cannot capture the raw proof, mark it
`UNVERIFIED`, never `FAIL`.**

- **Paste RAW evidence, not a paraphrase.** Every finding must include the actual API
  JSON or the actual Mongo doc (`db.<coll>.findOne(...)`). "It looked wrong" is not a finding.
- **Edit-unchanged check:** first `GET /chronicle/events/:iid`, copy the event's
  EXACT `ai_response` string, then PUT that exact string back. Do NOT retype/reconstruct
  it — a one-character difference counts as a real change and returns 200 (correctly).
- **Supersession / "deleted" memory check:** a superseded atom is `status:"superseded"`
  + `is_archived:true` and is HIDDEN from the default `GET /chronicle/memories` list.
  "Not in the active list" ≠ "deleted / not retired." Confirm in Mongo
  (`db.memories.findOne({_id})`) — check `status` and `superseded_by_event_ids` directly.
- **Cursor / location:** judge `current_location` against the LAST narrated place
  (the latest event's `location_anchor`), not against some earlier travel event.
- **Always check the DB for state claims**, the rendered surface can lag or filter.
- **Side-chat-leak test methodology (do it right or you'll false-positive):** reveal the
  secret ONLY inside `/side`. Do NOT type the secret (or ask about it by name) in a MAIN
  turn — the player's own typed words legitimately enter the prompt, so that "leak" is a
  test artifact. To prove a real leak: after the `/side` reveal, take a normal main turn
  that does NOT mention the secret, and check (a) the secret does NOT appear in main
  narration prose, and (b) the side character's **codex card** (`hidden_thought`,
  `mutable_state`, `immutable_facts`, `persona` — query `db.characters`) does NOT carry
  the secret (that card is injected verbatim into the main prompt). Only in a **sentient**
  world (protagonist ∉ `known_by`) does a surfaced secret = a real leak.

### 🟢 Known by-design — do NOT file these as bugs
- **GM-world side-chat surfacing.** In a GM world the player IS the protagonist, so a
  `/side` secret has the protagonist in its `known_by_entity_ids` and CORRECTLY appears
  in the protagonist's own Threads/Recap/Echoes (the gate intentionally passes it). A
  REAL side-chat leak must be shown in a **sentient** world (protagonist is NOT a
  knower → must be excluded) OR by a **different character/NPC** acting on knowledge
  they shouldn't have. Surfacing in the player's own Lore Tome in a GM world is not a leak.
- **`world_root_id: null`** on locations = the implicit single world. Not a bug.
- **Replay only on the latest turn** = product constraint, not a bug.

### Findings log
Append every flaw to a dated report **in `everlore-docs/playtest/`** (keep the doc
root clean) so the user/next agent can triage:
`everlore-docs/playtest/PLAYTEST_FINDINGS_<YYYY-MM-DD>__agent<N>.md`, one entry each:

```md
### [SEV: high|med|low] <one-line title>
- **World/instance:** <kind> "<title>" iid=<IID>
- **Repro:** exact messages/actions (or the agent-chat.ts invocation)
- **Expected vs got:** …
- **Evidence (RAW):** paste the actual API JSON / Mongo doc / log line that proves it
- **Verified how:** which DB doc or endpoint you confirmed against
- **Known gap?** link to a CHECKLIST/HANDOFF/merged-findings item, or "NEW"
```
Triage order: **silent data corruption > soft-lock / dead stream > wrong chip/POV > cosmetic.** Lead with corruption-class findings. For regression checks, state PASS/FAIL/UNVERIFIED with the raw proof inline.

## 8. Parallel agents — independent processes, each with its own world

The goal: N agents, each running as its **own process**, each creating its **own
world(s)**, playing **independently**, hunting flaws in parallel. That works — with
two facts to design around, both stemming from the **single shared dev account**.

### The two shared-account facts
1. **One WS pub/sub channel for the whole account** (`user:<uid>:events`). Every
   frame fans out to ALL of the account's open sockets. → **Each agent MUST filter
   frames by its own `instanceId`** (the harness does). Each agent opens its own WS
   connection; it'll see siblings' frames and must ignore them.
2. **Rate limits are per-account, i.e. GLOBAL across all agents** (`rate-limit.ts`):
   `template_create` **5/24h**, `chat` **10/60s**, `image_generate` 40/60min,
   `autofill` 30/60min. With many agents these are the real contention point.

### Isolation model (what "independent" means here)
- **Own instance = own world state.** The per-instance lock serializes a single
  instance; *different* instances run truly concurrently. Give every agent its own
  instance(s) — never two agents on one instance.
- **Own WS connection**, filtered by `instanceId` (above).
- **Own findings file** — `everlore-docs/playtest/PLAYTEST_FINDINGS_<date>__agent<N>.md`
  — so parallel writers don't clobber each other; merge at the end.
- Reuse ONE shared `TOKEN` across all agents (same account anyway) — avoids
  hammering the OTP endpoint and is simplest.

### Throughput + caps are ENV-TUNABLE (no source edits, no revert)
Four env vars on the **API + worker** processes raise the ceilings for a fleet.
Defaults are the safe production values, so you only set them for a QA run:

| Env var | Default | Set for a fleet | Controls |
|---|---|---|---|
| `GENERATION_CONCURRENCY` | 3 | ~10 | simultaneous turns the worker runs |
| `GENERATION_RATE_MAX` | 10 | ~60 | turns/min the worker accepts |
| `CHAT_RATE_MAX` | 10 | ~60 | player turns/60s (per account) |
| `TEMPLATE_CREATE_RATE_MAX` | 5 | ~100 | worlds created per 24h |

Launch the stack with them set (this is all "bumping that shit" requires now):
```bash
cd everlore-server
pkill -f "src/index.ts"; pkill -f "worker/index.ts"; sleep 1
export GENERATION_CONCURRENCY=10 GENERATION_RATE_MAX=60 CHAT_RATE_MAX=60 TEMPLATE_CREATE_RATE_MAX=100
nohup bun run src/index.ts    > /tmp/everlore-api.log    2>&1 &
nohup bun run worker/index.ts > /tmp/everlore-worker.log 2>&1 &
```
With these, each agent can create its OWN worlds and ~10 play concurrently without
queueing. (No code to revert — just relaunch without the exports for prod-like limits.)
You can still `redis-cli del rl:template_create:6a210ba38e6db660dc8ef6a3` to clear the
counter mid-run if needed.

Use `/templates/autofill` for variety so the fleet covers GM / sentient / character
× multiple genres (modern, fantasy, noir, slice-of-life), which surfaces
genre-specific bugs (calendar genre-fit, NSFW routing).

### Minimal fleet recipe
```bash
# 1. launch the stack with the env knobs above; grab one shared token (§1).
# 2. each agent creates its own world (§2) -> its own instance id ($IID).
# 3. spawn N independent player processes, each driving its own instance:
for IID in "$IID_A" "$IID_B" "$IID_C"; do
  TOKEN=$TOKEN bun run scripts/agent-chat.ts "$IID" \
    "Opening move in character." "/side <charId> A private word?" "/continue day" \
    > "/tmp/agent_${IID}.log" 2>&1 &
done
wait
# 4. each agent then reads its instance's Chronicle surfaces (§5) + logs findings (§7).
```
Each backgrounded run is an independent process driving its own instance; frame
filtering keeps them from cross-talking on the shared socket fanout.

---

## Reference: known product gaps this loop has surfaced

These are *expected findings* — if a run rediscovers them, they're already logged, not new:
- **Auth is mocked** (OTP `123456` works for any phone; JWT no expiry; default `JWT_SECRET`; WS token in query string) — not launch-ready. STILL OPEN.
- **`token_balance` is displayed but never read/decremented** — monetization is a no-op. STILL OPEN.
- **Free tier can't create** (creator/premium gated) — product decision, not a bug.

**Fixed June 12 2026 — verify they hold, don't re-file:**
- **Failure UX (server `1d52284`):** the turn lock is now heartbeated at a 90s TTL while a turn runs (was 240s the whole time), so a crashed/killed worker frees the player in ≤90s instead of ~4 min; an intermediate failure emits `generation_retrying` (turn still coming) before any `generation_failed`.
- **Creator friction (server `6222c00`):** creators can playtest their OWN unpublished world (non-owners get a clear 403); `template_create` budget is peeked-not-consumed (a failed create doesn't burn a slot) with `retryAfter`/`remaining` on the 429; `GET /templates/mine/quota` exposes the budget before the image-spending flow.
