# Agent Playtest Runbook — drive Everlore as the user, hunt for flaws

_How an autonomous agent logs in as the account owner, creates worlds (with cover
images), starts playthroughs, chats over WebSocket, exercises the whole product,
and reports real bugs + the known gaps. This is **self-testing of the owner's own
account on the local/dev stack** — authorized QA, not third-party access._

> Companion docs: `LIVE_VERIFICATION.md` (driving the worker directly, the deeper
> "Tier-2" path + false-positive traps), `HANDOFF.md` / `CHECKLIST.md` (what's
> built), `LOCATION_GRAPH.md` (place model). This runbook is the **product-level
> Tier-3-ish loop**: act like a player through the real API.

---

## 0. Golden rules (read first)

1. **Keeper worlds are off-limits for destructive tests.** The 6 sample worlds +
   "The Unseen Child" (`6a2869768f7446e38bdb6fce`) are deliberately kept. Create
   your OWN throwaway worlds/instances for anything you'll mutate, and delete them
   when done. If you must drive a keeper instance, rewind it back to baseline after
   (see §7).
2. **Restore state after out-of-band writes.** Any time you write to an instance
   outside the normal WS turn (rewind, direct Mongo edit, repair script), you MUST
   bust the session cache: `redis-cli del session:<instanceId>`. Stale
   `session:<iid>` is the #1 false-positive trap.
3. **No watch mode.** The API/worker do NOT auto-reload. After ANY server code edit,
   restart them (see §1) or you're testing stale code.
4. **One turn per instance at a time.** A per-instance lock (`lock:gen:<player>:<iid>`)
   serializes turns. To parallelize, give each agent its OWN instance — never two
   agents hammering one instance.
5. **Mind the rate limits** (`src/middleware/rate-limit.ts`): `chat` 10/60s,
   `template_create` **5/24h**, `image_generate` 40/60min, `autofill` 30/60min.
   The template-create wall is the tight one — budget your world creation.
6. **Log every flaw** as you find it (see §8). A run that finds bugs but doesn't
   record them is wasted.

---

## 1. Environment up

```bash
cd /Users/fatman/Desktop/codes/rpg-ai/everlore-server
# Mongo + Redis must be running locally first (the app assumes localhost defaults).
pkill -f "src/index.ts"; pkill -f "worker/index.ts"     # clean slate
nohup bun run src/index.ts    > /tmp/everlore-api.log    2>&1 &
nohup bun run worker/index.ts > /tmp/everlore-worker.log 2>&1 &
# Wait for health:
for i in $(seq 1 15); do [ "$(curl -s -o /dev/null -w '%{http_code}' localhost:3000/health)" = "200" ] && { echo API_UP; break; }; sleep 1; done
tail -3 /tmp/everlore-worker.log   # expect "Everlore Worker Cluster running"
```

- API: `http://localhost:3000`  •  WS: `ws://localhost:3000/ws/play?token=<jwt>`
- Logs: `/tmp/everlore-api.log`, `/tmp/everlore-worker.log` (watch these for the
  real story — stack traces, `generation_failed`, `continuity.drift`).

---

## 2. Log in as the owner (mocked OTP on dev)

Twilio is unconfigured on dev, so OTP is **mocked**: any phone + code `123456`.

```bash
PHONE='+19474877175'        # the owner account (tier: creator, nsfw_enabled: true)
curl -sX POST localhost:3000/auth/otp/send   -H 'content-type: application/json' -d "{\"phone\":\"$PHONE\"}"
TOKEN=$(curl -sX POST localhost:3000/auth/otp/verify -H 'content-type: application/json' \
  -d "{\"phone\":\"$PHONE\",\"code\":\"123456\"}" | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
echo "${TOKEN:0:16}…"
curl -s localhost:3000/auth/me -H "authorization: Bearer $TOKEN" | python3 -m json.tool
```

> The JWT has **no expiry** on dev (a known gap — see §9), so one token lasts the
> whole run. Pass it as `Authorization: Bearer <TOKEN>` on REST and as the
> `?token=` query param on the WebSocket.

---

## 3. Create a world (the three world types)

`POST /templates` then `POST /templates/:id/publish`. **Instances require a
PUBLISHED template** (a creator can also playtest their OWN unpublished one — see
§5). Required body fields: `title`, `description`, `seed_prompt` (≥10 chars),
`global_lore` (may be ""), `is_sentient`, `is_nsfw_capable`. **Worlds (`kind:'world'`)
also need ≥1 stat in `base_stats_template`**; character-kind may have zero.

| Want | Set |
| --- | --- |
| **GM world** (player has a protagonist, third-person narration) | `kind:'world'`, `is_sentient:false` |
| **Sentient world** (player speaks AS a persona to an AI character) | `kind:'world'`, `is_sentient:true` |
| **Character companion** (single companion, first-person) | `kind:'character'` |

```bash
BODY='{
  "title":"QA Probe World",
  "kind":"world",
  "is_sentient":false,
  "is_nsfw_capable":false,
  "description":"throwaway world for agentic playtesting",
  "seed_prompt":"A rain-slick neon harbor district at midnight. The player is a courier who just found a package they were not supposed to open.",
  "global_lore":"The harbor is run by three rival syndicates.",
  "base_stats_template":{"nerve":{"default":50,"min":0,"max":100,"description":"composure under pressure"}},
  "scene_tags":["dialogue","tension"]
}'
TID=$(curl -s -X POST localhost:3000/templates -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' -d "$BODY" \
      | python3 -c 'import sys,json;print(json.load(sys.stdin)["_id"])')
echo "templateId=$TID"
curl -s -X POST localhost:3000/templates/$TID/publish -H "authorization: Bearer $TOKEN" -o /dev/null -w "publish=%{http_code}\n"
```

**Tip:** to draft a richer world fast, use `POST /templates/autofill`
(`{"target":"world","brief":"...","is_sentient":false}`) — it returns a full draft
you can post to `/templates`. Good for variety across many test worlds.

### Check your creation budget BEFORE spending on images
```bash
curl -s localhost:3000/templates/mine/quota -H "authorization: Bearer $TOKEN"
# {"can_create":true,"reason":null,"remaining":4,"retry_after":null}
```
If `can_create:false` you're at the 5/24h wall — don't burn image calls crafting a
world you can't submit.

---

## 4. Give the world a cover image (optional but part of "full system")

Image generation is **creator/premium only** (the owner is `creator`). It calls
Seedream → uploads to S3/CDN and returns a URL.

```bash
IMG=$(curl -s -X POST localhost:3000/templates/image/generate -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"prompt":"a rain-slick neon harbor at midnight, cinematic, moody"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["url"])')
echo "image=$IMG"
# Attach it to the template:
curl -s -X PUT localhost:3000/templates/$TID -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d "{\"image_url\":\"$IMG\"}" -o /dev/null -w "attach=%{http_code}\n"
```
> Flaw-hunt here: does a failed/slow Seedream call surface a clean error, or hang?
> Is the returned URL actually reachable (CDN)? Re-rolls cost real money — note if
> there's no budget signal before spend.

---

## 5. Start a playthrough (instance)

```bash
IID=$(curl -s -X POST localhost:3000/instances -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d "{\"template_id\":\"$TID\"}" | python3 -c 'import sys,json;d=json.load(sys.stdin);print((d.get("instance") or d)["_id"])')
echo "instanceId=$IID"
```
- **GM worlds** want a protagonist name:
  `POST /instances/:id/protagonist  {"name":"Wren","identity":"a courier with a debt"}`.
- **Playtest path:** a creator can start an instance on their OWN *unpublished*
  template (non-owners get a clear 403). Useful for testing before publish.
- Reset a playthrough: `POST /instances/:id/reset`. Archive: `POST /:id/archive`.

---

## 6. Chat — the WebSocket contract (this is the whole game loop)

**Chat is WS-only — there is NO REST chat endpoint.** Connect to
`ws://localhost:3000/ws/play?token=<jwt>`, wait for the `connected` frame, then send
actions. Every inbound field must be on the schema (`action`, `instance_id`,
`event_id`, `payload`).

| Action | Frame | Notes |
| --- | --- | --- |
| chat | `{action:"chat", instance_id, payload:{message}}` | a normal turn |
| continue | `{action:"continue", instance_id, payload:{advance?}}` | world moves on its own; `advance` ∈ hours\|day\|days\|season = time skip |
| side_chat | `{action:"side_chat", instance_id, payload:{character_id, message}}` | private 1:1; story time/cursor FROZEN |
| replay | `{action:"replay", instance_id, event_id}` | regenerate an existing turn |
| ping | `{action:"ping"}` | keepalive → `pong` |

**Outbound frames for a chat turn (in order):**
`connected` → `ack` → `generation_delta`×N (streamed tokens) →
`generation_stream_end` → `generation_complete` (carries `choices`,
`present_characters`, `location_anchor`, stat diffs). Then async, on their own:
`character_codex_updated`, `memories_curated`.
**Failure frames:** `generation_retrying` (transient, turn still coming),
`generation_failed` (gave up), `error` (e.g. `code:GENERATION_IN_PROGRESS`,
`RATE_LIMITED`). Side-chat: `side_chat_delta`/`side_chat_complete`/`side_chat_error`.
Replay: `replay_delta`/`replay_complete`.

### Drop-in WS driver (`scripts/agent-chat.ts`)
Run with `bun run scripts/agent-chat.ts <TOKEN> <IID> "your message"`. Prints the
streamed reply + the structured tail, exits when the turn completes or fails.

```ts
const [token, iid, ...rest] = process.argv.slice(2)
const message = rest.join(" ") || "I look around and take stock of the situation."
const ws = new WebSocket(`ws://localhost:3000/ws/play?token=${token}`)
let prose = ""
const done = (code: number) => { try { ws.close() } catch {} process.exit(code) }
const timeout = setTimeout(() => { console.error("TIMEOUT — no completion in 90s"); done(2) }, 90_000)

ws.onmessage = (ev) => {
  const m = JSON.parse(ev.data as string)
  switch (m.type) {
    case "connected": ws.send(JSON.stringify({ action: "chat", instance_id: iid, payload: { message } })); break
    case "ack": break
    case "generation_delta": prose += m.delta ?? ""; process.stdout.write(m.delta ?? ""); break
    case "generation_stream_end": break
    case "generation_retrying": console.error(`\n[retrying ${m.attempt}/${m.maxAttempts}]`); break
    case "generation_complete": {
      const e = m.event ?? {}
      console.log("\n\n— CHOICES:", JSON.stringify(e.choices ?? []))
      console.log("— PRESENT:", JSON.stringify(e.present_characters ?? []))
      console.log("— LOCATION:", JSON.stringify(m.location_anchor ?? e.location_anchor ?? null))
      console.log("— SEQ:", e.sequence)
      clearTimeout(timeout); done(0); break
    }
    case "generation_failed": console.error("\nFAILED:", m.message); clearTimeout(timeout); done(1); break
    case "error": console.error("\nERROR:", JSON.stringify(m)); clearTimeout(timeout); done(1); break
  }
}
ws.onerror = (e) => { console.error("WS error", e); done(1) }
```

Multi-turn: keep one socket open and send the next `chat` only after the previous
`generation_complete` (the lock enforces this anyway). For a fuller conversation,
loop messages that react to the returned `choices`.

---

## 7. Restore / cleanup discipline

- **Delete throwaway worlds you created:**
  `DELETE /templates/:id` (cascades the template's instances) — verify the instance
  is gone too (it cascades via `deletionService`).
- **Rewind a keeper instance you drove** back to baseline (don't delete it):
  ```bash
  cd /Users/fatman/Desktop/codes/rpg-ai/everlore-server
  bun -e 'import {connectMongo} from "./src/config/mongo";import {connectRedis,getRedisClient} from "./src/config/redis";
  import {memoryService} from "./src/services/memory.service";
  const [iid,pid,base]=["<IID>","6a210ba38e6db660dc8ef6a3", <BASE_SEQ>];
  await connectMongo();await connectRedis();
  await memoryService.rewindToSequence(iid,pid,base+1);
  await getRedisClient().del(`session:${iid}`); console.log("restored to",base); process.exit(0)'
  ```
  (The Unseen Child baseline is **seq 26**.)
- **Always** `redis-cli del session:<iid>` after any out-of-band write.
- **Don't leave the owner's `token_balance` or rate-limit counters in a surprising
  state.** If you reset `rl:template_create:<uid>` for a test, restore it.
- Note for the next reader whether you left the servers running.

---

## 8. What to look for — the flaw-hunting checklist

### A. The KNOWN gaps (confirm these specifically — from CHECKLIST/HANDOFF/LOCATION_GRAPH)
- **Tier-3 visual/UX** (the main open gap — but you're API-level; still watch the
  *data* the UI would render): are `choices` always first-person from the
  protagonist's POV? Is `present_characters` correct (no one flickering to
  "Elsewhere", no one bleeding in after they left)?
- **Location state (probabilistic):** do a turn that visibly transforms a place
  (restore a ruin, smash a window) — does `location_state` update? (Often under-fires.)
- **Memory version-linking (probabilistic):** state a fact, then later contradict it
  — does the old memory get superseded/linked? (Gated on a 0.82 match; may not fire.)
- **Travel + calendar:** move between named places → expect a `travel` event +
  cursor move; narrate "weeks pass" → expect the story date to advance.
- **Side-chat isolation:** a private side_chat must NOT advance story time/location,
  and its secrets must NOT leak into main narration (unless shared in the main story).
- **Same-name place collision** (only testable once a world has 2 towns): a
  "tavern" in town A must stay distinct from a "tavern" in town B.

### B. GENERAL flaw-hunting (the open-ended part — this is where new bugs live)
- **Failure UX:** kill the worker mid-turn (`pkill -f worker/index.ts`) — does the
  player get freed within ~90s (lock heartbeat) and a `generation_failed`/retry, or
  soft-locked? Send two turns fast — clean `GENERATION_IN_PROGRESS`?
- **Continuity:** after ~15–30 turns, run the continuity audit (below) — any
  warn/fail? Contradictions in names, relationships, places, dates?
- **Identity/POV drift:** in a GM world, does the narrator ever address the player
  in the wrong person, or invent a duplicate of the protagonist?
- **Codex hygiene:** does a vague "the man"/"the stranger" spawn a junk character
  card? Do duplicate cards appear for one person?
- **Prompt-injection / boundary:** try a message that tries to rewrite the rules,
  reveal the system prompt, or break NSFW gating — does it hold?
- **Recovery:** disconnect the WS mid-stream and reconnect / `load_instance` — is the
  turn recoverable or lost?
- **Economy/limits:** confirm `token_balance` is never actually decremented (known),
  and that hitting `chat` 10/min returns a clean `RATE_LIMITED` with `retryAfter`.
- **Chronicle surfaces:** after play, read each and sanity-check it reflects the
  story: `GET /chronicle/recap/:iid`, `/timeline/:iid`(if present),
  `/calendar/:iid`, `/locations/:iid`, `/relationships/:iid`, `/threads/:iid`,
  `/memories?instance_id=:iid&q=...`.

### C. The audit toolkit (code-health, run anytime)
```bash
cd /Users/fatman/Desktop/codes/rpg-ai/everlore-server
bun run typecheck
bun run audit:memory-links audit:location audit:location-resolution \
        audit:movement audit:time-skip audit:codex-dedup audit:choices audit:replay-edit
bun run scripts/rewind-audit.ts
# Per-instance continuity (read-only):
curl -s "localhost:3000/admin/instances/$IID/continuity-audit" -H "authorization: Bearer $TOKEN" | python3 -m json.tool
```

---

## 9. Standing product gaps (don't re-report these as "new" — they're known)
- **Auth not launch-ready:** OTP mocked (`123456`), JWT has no expiry,
  `JWT_SECRET` defaults to a dev string, WS token rides the query string.
- **Monetization non-functional:** `token_balance` is shown but never read/decremented.
- **Free tier can't create** (creator/premium gated) — product decision, not a bug.
- Failure-UX server hardening + creator-friction (playtest unpublished, peek-don't-
  consume quota, `/templates/mine/quota`) were FIXED June 12 — verify they still hold,
  don't re-file.

---

## 10. Logging findings

Append every finding to a dated report so the next agent/the user can triage:
`everlore-docs/PLAYTEST_FINDINGS_<YYYY-MM-DD>.md`. One entry per finding:

```md
### [SEV: high|med|low] <one-line title>
- **World/instance:** <kind> "<title>" iid=<IID> (throwaway? yes/no)
- **Repro:** exact messages/actions sent (or the script invocation)
- **Expected vs got:** …
- **Evidence:** frame dump / log excerpt (`/tmp/everlore-*.log`) / audit output
- **Known gap?** link to CHECKLIST/HANDOFF item, or "NEW"
```

Triage rule of thumb: **silent data corruption > soft-lock / dead stream > wrong
chip/POV > cosmetic.** Lead the report with the corruption-class findings.

---

## 11. Running several agents at once
- Each agent: **its own throwaway instance** (the per-instance lock serializes a
  single instance; separate instances run truly in parallel).
- Shared limits to respect across all agents: `template_create` 5/24h is **global to
  the account** — coordinate world creation, or have ONE agent create a batch up front
  and the rest reuse instances of them.
- `chat` 10/60s is per-user (also global) — stagger turns if many agents chat at once,
  or you'll see `RATE_LIMITED`.
- The worker runs `concurrency:3` (limiter 10/min) — more than ~3 simultaneous turns
  queue; that's fine, just slower.
```
