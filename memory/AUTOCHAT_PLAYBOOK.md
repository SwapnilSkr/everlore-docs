# Auto-Chat QA Playbook

**What this is:** the exact, repeatable procedure the QA sub-agents used to (1) create worlds as the real user, (2) auto-play them over the live WebSocket contract, and (3) surface product flaws. Hand this to any agent and it can reproduce the whole loop with no extra context.

**Audience:** an autonomous agent with Bash + the running Everlore server. Everything here runs against the **local dev stack** (`localhost:3000`). Do not point it at anything else.

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

### 2a. (optional) Generate a cover image first
```bash
IMG=$(curl -sX POST "$BASE/templates/image/generate" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  -d '{"prompt":"rain-slicked neon alley, cyber-noir, cinematic, moody"}' | jq -r '.image_url')
```
Returns a CDN URL. Image gen costs money and runs **before** any validation — generate only when the template body is otherwise ready.

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
- **Send a turn:** `{"action":"chat","instance_id":"<INST>","payload":{"message":"..."}}`
- **Inbound actions** you can send: `chat`, `continue` (`payload.advance` = `hours|day|days|season` to skip time), `side_chat` (`{character_id,message}` — private aside to one NPC), `replay`, `load_instance`, `ping`.

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
logs in as the dev owner (or reuses a `TOKEN` env var), plays each message in
sequence on one socket, streams the prose, and dumps the structured tail of every
turn so flaws are visible.

```bash
cd everlore-server
bun run scripts/agent-chat.ts <INSTANCE_ID> "first message" "second message" ...
# or reuse a token:  TOKEN=<jwt> bun run scripts/agent-chat.ts <INSTANCE_ID> "..."
```

Per turn it prints: `NARRATIVE` (streamed prose), then
`[seq N] scene=… tone=… time=…`, `location`, `present`, `choices`, plus async
`[codex]` / `[memory]` lines as they land. It waits for `generation_complete`
before sending the next turn (the per-instance lock requires this anyway) and
leaves a 1.5 s gap so the async codex/memory projections land first.

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
8. **Side-chat privacy** — `side_chat` an NPC with a secret, then in the main thread confirm the secret didn't leak into world-visible narration.
9. **NPC codex** — after introducing an NPC, confirm a `character_codex_updated` fired and no duplicate/"Mysterious Man" phantom card was minted.
10. **Failure UX** — (optional, destructive) kill the worker mid-turn and confirm the client surfaces a failure rather than soft-locking. Restart the worker after.

Record, per world: which probes were **GREEN** vs the exact payload that was wrong. That delta is the deliverable.

---

## 5. Read the projections (audit surfaces)

After playing, verify the side-effects, not just the prose:

- **Chronicle read endpoints** (REST, same bearer token) — recap, timeline, echoes, **almanac** (calendar), places, bonds, threads. Browse `everlore-server/src/routes/chronicle.routes.ts` for exact paths.
- **Memory atoms / entity graph** — query Mongo directly: `db.memories.find({instance_id: <oid>})`, `db.entities.find({instance_id: <oid>})`.
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
2. **Record every dev-state mutation** in your handoff: created template/instance ids, any `token_balance` bump, any `rl:template_create` reset, whether you left servers running.
3. The new worlds + their ids belong in the session handoff so the next agent knows they're intentional keepers, not test litter.

### Findings log
Append every flaw to a dated report so the user/next agent can triage:
`everlore-docs/PLAYTEST_FINDINGS_<YYYY-MM-DD>.md`, one entry each:

```md
### [SEV: high|med|low] <one-line title>
- **World/instance:** <kind> "<title>" iid=<IID> (throwaway? yes/no)
- **Repro:** exact messages/actions (or the agent-chat.ts invocation)
- **Expected vs got:** …
- **Evidence:** frame dump / log excerpt (/tmp/everlore-*.log) / audit output
- **Known gap?** link to a CHECKLIST/HANDOFF item, or "NEW"
```
Triage order: **silent data corruption > soft-lock / dead stream > wrong chip/POV > cosmetic.** Lead with corruption-class findings.

## 8. Running several agents at once
- Give each agent its **own throwaway instance** — the per-instance lock serializes one instance, so separate instances run truly in parallel.
- `template_create` (5/24h) and `chat` (10/60s) limits are **per-account = global across all your agents.** Coordinate: have ONE agent create a batch of worlds up front, the rest reuse instances; stagger turns or you'll hit `RATE_LIMITED`.
- The worker is `concurrency:3` (limiter 10/min) — more than ~3 simultaneous turns just queue (slower, not broken).

---

## Reference: known product gaps this loop has surfaced

These are *expected findings* — if a run rediscovers them, they're already logged, not new:
- **Auth is mocked** (OTP `123456` works for any phone; JWT no expiry; default `JWT_SECRET`; WS token in query string) — not launch-ready. STILL OPEN.
- **`token_balance` is displayed but never read/decremented** — monetization is a no-op. STILL OPEN.
- **Free tier can't create** (creator/premium gated) — product decision, not a bug.

**Fixed June 12 2026 — verify they hold, don't re-file:**
- **Failure UX (server `1d52284`):** the turn lock is now heartbeated at a 90s TTL while a turn runs (was 240s the whole time), so a crashed/killed worker frees the player in ≤90s instead of ~4 min; an intermediate failure emits `generation_retrying` (turn still coming) before any `generation_failed`.
- **Creator friction (server `6222c00`):** creators can playtest their OWN unpublished world (non-owners get a clear 403); `template_create` budget is peeked-not-consumed (a failed create doesn't burn a slot) with `retryAfter`/`remaining` on the 429; `GET /templates/mine/quota` exposes the budget before the image-spending flow.
