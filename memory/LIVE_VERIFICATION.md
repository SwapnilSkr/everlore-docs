# Live-turn verification — runbook

_How to actually stress-test Everlore's LLM-dependent behaviour against real
generated turns, without producing false positives. Written after a June-2026 pass
that drove real turns through the worker and caught **5 real bugs the unit audits
could not** (3 in the P2.6 movement/presence work, 2 in Phase 6B time/location-state).
Read this before claiming any LLM-path feature "works" — `typecheck` + audits passing
is necessary but NOT sufficient._

## Why this exists

Most Everlore correctness lives at a **witness seam**: a small model (the scene
extractor / codex extractor) emits signals, and the server turns them into state
(cursor, presence, calendar, memories, codex). Unit audits exercise the *pure server
math* and the *extractor in isolation* — they are cheap and trustworthy, but they
**cannot** see:

- what the live extractor actually returns on real narrator prose (it under-reports;
  e.g. `viewpoint_moved`, `time_elapsed`, `location_state_changes` all silently miss);
- the **interaction** of the generation processor + async memory curation + caches +
  the calendar + dedup, end to end;
- **input-vs-output mismatches** — the extractor reads only the AI narrative, so a
  signal the *player* wrote ("Weeks pass.", "I go to my room") that the narrator
  doesn't restate is lost.

Every bug in the list below was invisible to green audits and only appeared when a
real turn ran. So: **a feature is not verified until a live turn has exercised it and
the resulting Mongo state has been inspected.**

## The three tiers (do them in order)

1. **Tier 1 — pure/cheap audits.** `bun run audit:*` (movement, time-skip, location,
   location-resolution, choices, codex-dedup, replay-edit). No DB or LLM-only on a
   throwaway clone. Run these FIRST and after every change — they're fast and
   deterministic. They prove the *server math* and *extractor-in-isolation*, not the
   live pipeline.
2. **Tier 2 — live worker turns (THIS DOC).** Drive real generated turns through the
   worker against a dev instance, inspect the persisted state, then restore. This is
   where the real bugs hide.
3. **Tier 3 — in-app UI.** The user's manual pass in the Flutter app (chip render/tap,
   presence tags, journal/atlas, side-chat stream). Not scriptable here; hand off with
   a concrete "play X, expect Y" list.

## Prerequisites

- `everlore-server/.env` present with `MONGODB_URI`, `REDIS_URL`, `OPENAI_API_KEY` /
  `OPENROUTER_API_KEY`, `NARRATION_*_MODEL`. (`redis-cli ping` → `PONG`.)
- A dev instance with history. The June-2026 dev instance: `instanceId
  6a2869768f7446e38bdb6fce`, `playerId 6a210ba38e6db660dc8ef6a3` (a single-mansion GM
  world). Confirm its current `max(sequence)` and cursor before you start.
- **No HTTP server / WebSocket is needed.** The worker streams deltas via
  `redis.publish` (fire-and-forget); with no subscriber the deltas are dropped but the
  **event still persists**. You only need the worker + Mongo + Redis.

## The harness recipe

1. **Start the worker** (NOT `--watch`; you'll restart it manually after code edits):
   ```bash
   bun run worker/index.ts > /tmp/everlore-worker.log 2>&1 &
   # wait for: "Everlore Worker Cluster running"
   ```
2. **Record a baseline**: `max(sequence)`, `current_location`, `current_time_anchor`,
   `count(entities type:location)`. This is your restore point and your diff base.
3. **Enqueue a real turn** from a throwaway script (the same calls the WS handler makes):
   - main turn: `generationService.dispatch({ instanceId, playerId, userMessage })`
   - side chat: `generationService.dispatchSideChat({ instanceId, playerId, characterId, userMessage })`
   - The script must `await connectMongo()` **and** `await connectRedis()` (the queue +
     `loadSession` both use Redis — forgetting Redis throws "Redis not connected").
4. **Poll Mongo** for the new event (`sequence > baseSeq`), then **wait ~2–4 s more**
   for async projections (see Trap C) before asserting.
5. **Inspect**: the new event's `type`, `location_anchor`, `data.present_characters`,
   `data.travel`, `data.time_advanced`; the instance `current_location` +
   `current_time_anchor`; the location-entity list (watch the count for stray mints);
   minted memories (`origin`, `known_by_entity_ids`, `location_*`).
6. **Restore** (see Cleanup): `memoryService.rewindToSequence(iid, pid, baseSeq+1)` then
   `redis.del('session:<iid>')`, plus any out-of-band entity reset.

## FALSE-POSITIVE TRAPS — read this twice

These each produced a WRONG reading during the June pass. Every one is a way to "prove"
something works (or is broken) when the truth is the opposite.

- **Trap A — stale session cache.** `instanceService.loadSession` caches
  `session:<iid>` in Redis for **1 hour**. Any **out-of-band Mongo write** to the
  instance (a repair script, a manual cursor/entity edit, a `rewind` done by raw query)
  is **invisible to the next turn** until the cache expires. This made a bedroom→dining
  turn report a backwards `travel{from:"dining room"}` (the pre-repair cursor). **RULE:
  `redis.del('session:<iid>')` after ANY out-of-band instance write**, and the repair
  scripts now do this — verify they did.
- **Trap B — the worker runs the OLD code.** `bun run worker/index.ts` is not watched.
  After editing anything under `worker/` or `src/` that the worker imports, you MUST
  `pkill -f "worker/index.ts"` and restart, or you're testing stale code. (A "fix that
  didn't take" is almost always this.)
- **Trap C — async projections lag the event.** The event persists synchronously, but
  **memory curation, `location_state`/`location_facts` application, codex deltas, and
  summaries run on SEPARATE queues afterward.** Asserting "no memory minted" / "no
  location_state" immediately after the event appears gives a false negative. Wait and
  re-poll (curation took ~3–8 s in the dev instance). Conversely, a stray entity may
  appear only *after* the curation pass — re-check the entity count post-curation.
- **Trap D — the extractor sees ONLY the AI narrative, never the player input.** If a
  turn's intent lives in the player's text ("Weeks pass", "I go to my room") and the
  narrator's prose doesn't restate it, the extractor cannot emit it — and that's not an
  extractor bug, it's a *missing signal* the server must backstop from `player_input`
  (this is what `movement-signal.ts` / `time-skip-signal.ts` do). When a field comes
  back empty, ask "was the signal even in the prose the extractor saw?" before blaming
  the extractor.
- **Trap E — the LLM is stochastic; one run is not a result.** A field that comes back
  empty once may populate on the next call. For extractor-dependent assertions, **run
  the extractor directly 3×** (the `audit:*` scripts show the invocation) on the actual
  prose to tell a *reliable* miss from a *one-off*. Probabilistic fields
  (`location_state_changes`) are **not pass/fail on a single turn** — report them as
  "improved N/M", not "works".
- **Trap F — isolate model-miss from apply-path-bug.** When state is wrong, determine
  WHICH layer failed: run `extractSceneMetadata(prose, …)` directly to see what the
  model emitted, then check whether the processor applied it. "Calendar didn't advance"
  was a *missing signal* (Trap D), not a broken `advanceDays`. Don't fix the wrong layer.
- **Trap G — the cursor follows `current_location` regardless of `viewpoint_moved`.** A
  stuck cursor does NOT necessarily mean the move signal failed — the cursor follows
  whatever place the model named. Check the model's `current_location`, not just the
  boolean, before concluding.
- **Trap H — you are mutating a real save.** Live turns append real events/memories/
  projections to the user's instance. Always restore (Cleanup) unless the user wants the
  turns kept. Confirm the final state matches the baseline.

## Stress-test matrix (what to actually exercise)

Don't test one happy path. For each subsystem, drive the **adversarial** cases too. Good
turns to send (GM world, protagonist = the player):

**Movement / location (P2.6 + Location Graph)**
- move to a named place ("I head to the garden") → cursor follows, `travel` marker, presence resets, **no duplicate entity** (reuse on return).
- retreat to an OWNED room ("I go to my room and shut the door") → cursor → "<owner>'s room", presence `[]` (locals don't follow), entity reused.
- LEAVE origin vs go to destination ("I leave my room and head to the dining room") → must land on the DESTINATION, not invert (the origin-room trap).
- stay-put with a feint ("…thinking about going to the garden tomorrow") → `narration`, NO move, NO phantom travel, cursor unchanged.
- open-world scale ("I travel to the city of X", "I cross into the Shadow Realm") → witness names it, cartographer places under a world-root.
- vague labels ("the room", "outside", "his room") → never mint a new entity (cursor path AND the memory-curation path; check the entity count).

**Presence**
- a quiet unnamed character at the table stays present (carry-forward); a character the prose says leaves drops (`characters_departed`); on a move, locals reset.
- KNOWN GAP (don't flag as bug): traveling companions reset on a move (no `travelling_with` model yet) — see LOCATION_GRAPH "Open-world limits".

**Time / calendar (Phase 6B)**
- explicit skip in player input ("Weeks pass.") even if the narrator drops it → calendar advances (the time-skip backstop).
- named period ("the next morning") → +1 day; non-skip ("for years I've wanted this", "I train every day") → NO advance.

**Location state/facts (Phase 6B)**
- a positive transformation ("I restore the garden") AND a destructive one ("the gate is smashed") → `location_state` captured (probabilistic; sample a few).

**Side-chat (Phase 7)**
- dispatch a private chat → in-character reply, **story time + cursor FROZEN**, the `side_chat` event excluded from main reads, memory `origin:'side_chat'` + correct `known_by_entity_ids` (GM world includes the protagonist; a sentient/other-character secret must NOT reach main retrieval).

**Dedup / rewind safety**
- return to a place via a variant name ("the garden" when the world knows "Night Garden") → resolves to the SAME entity. `rewindToSequence` removes the test turns + their memories cleanly.

## Cleanup / leave-no-trace

1. `memoryService.rewindToSequence(iid, pid, firstTestSeq)` — removes that seq and
   everything after, recomputes cursor + projections, prunes sourced memories. (Handles
   `side_chat` events too — they share the ledger.)
2. `redis.del('session:<iid>')` — Trap A.
3. Reset any entity you mutated out-of-band during the test (e.g. a `location_state` a
   verification turn minted, if you're restoring to before it).
4. `pkill -f "worker/index.ts"`.
5. `rm` throwaway scripts; `rm /tmp/everlore-worker.log`.
6. **Verify** the final state == baseline (max seq, cursor, calendar, entity count,
   `origin:'side_chat'` memory count). Don't trust the rewind — check it.

## Interpreting results honestly

- **Deterministic server math** (presence fold, cursor-follow, dedup, rewind) → pass/fail
  on a single run. A miss is a real bug.
- **Witness-dependent fields** (movement, time, location-state, choices, codex) → the
  small model is unreliable; distinguish "model didn't emit" (Trap D/F) from "server
  mishandled what it emitted". Fix the right layer. Prefer a **deterministic server
  backstop from `player_input`** over leaning on the prompt (the proven pattern:
  `movement-signal.ts`, `time-skip-signal.ts`, the presence fold).
- **Probabilistic fields** → report as a rate, never "works" from one turn.

## Known NON-bugs (do not chase)

- Presence **over-persistence** (a quietly-vanished character lingers until a scene
  break) — accepted gentle failure.
- **Partial ancestry** (`dining room → mansion → ?`) — valid fog-of-war.
- `location_state` occasionally missed — probabilistic; mitigated by prompt, not solved.
- **Open-world limits** (intra-world same-name collision, traveling-party presence,
  dedup-at-scale, mobile containers, parallel scenes) — documented + deferred in
  `LOCATION_GRAPH.md` "Open-world limits"; the collision one is a real latent bug but
  content-gated, the rest are content-driven.

## Appendix — reusable script skeletons

**Live main turn** (`scripts/_live-turn.ts`, throwaway):
```ts
import { connectMongo, coll } from '../src/config/mongo'
import { connectRedis } from '../src/config/redis'
import { generationService } from '../src/services/generation.service'
import { ObjectId } from 'mongodb'
const IID='6a2869768f7446e38bdb6fce', PID='6a210ba38e6db660dc8ef6a3'
const MSG = process.argv[2]
await connectMongo(); await connectRedis()
const oid = new ObjectId(IID)
const base = (await coll('events').find({instance_id:oid}).sort({sequence:-1}).limit(1).toArray())[0]?.sequence ?? 0
await generationService.dispatch({ instanceId:IID, playerId:PID, userMessage:MSG })
// poll events for sequence>base, then setTimeout ~3s for async projections, then inspect
```

**Direct extractor probe** (isolate Trap D/E/F — what did the MODEL emit?):
```ts
import { extractSceneMetadata } from '../worker/lib/metadata-extractor'
for (let i=0;i<3;i++) {
  const m = await extractSceneMetadata(prose, [], [], {
    isSentient:false, currentLocationName:'garden', priorPresent:[],
    protagonist:{name:'Swapnil Sarkar',aliases:[]}, roster:[], knownPlaces:[{name:'garden',aliases:[]}],
  } as any)
  console.log(i, m.time_elapsed, m.location_state_changes, m.viewpoint_moved, m.current_location)
}
```

**Restore**:
```ts
await memoryService.rewindToSequence(IID, PID, base+1)
await getRedisClient().del(`session:${IID}`)
```

## Track record (June 2026 pass)

Bugs this method caught that green audits missed: (P2.6) possessive namer grabbed the
ORIGIN room; `isVagueLocationLabel` missed possessive-pronoun rooms → memory-curation
minted ghost atlas nodes; repair script left a stale session cache. (Phase 6B)
time-advance signal lost between player input and the extractor; `location_state`
under-detected positive transformations. All fixed; see `CHECKLIST.md` + `HANDOFF.md`.
