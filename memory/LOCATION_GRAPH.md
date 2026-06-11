# Location Graph — precise, nested, world-scoped places

_Implementation spec. Operationalizes the location vision sketched in
`CALENDAR_TIMELINES_AND_LOCATIONS.md` with the decisions locked in the June-2026
design discussion. Read that doc first for the wider time/place picture; this doc
is the concrete data model + server logic + rollout for a duplication-proof,
nested, infinitely-extensible location graph._

## Why

Locations are not cosmetic — they are a **first-class retrieval signal**. Memories
carry a place label (`location_entity_id` / `location_anchor` / `location_name`,
[memory.model.ts](../../everlore-server/src/models/memory.model.ts)), and
`queryRag` runs a dedicated **place-recall arm** ("what happened *here* before?",
[rag.provider.ts](../../everlore-server/src/providers/rag.provider.ts)) fused with
the vector/keyword/entity/thread arms. So **the precision of the location graph IS
the precision of place-scoped lore.** Two failure classes follow directly:

1. **Fragmentation** — a vague "the room" minted alongside "dining room" splits the
   dinner's memories across two nodes; place-recall then misses half. (Repaired
   once already via `merge:location`; the rules below stop it recurring.)
2. **Collision / bleed** — if "the house" in City A and "the house" in City B ever
   resolve to ONE entity, the place-recall arm pulls City-A events while the player
   stands in the City-B house → hallucination. The current fuzzy resolver
   (`a12e94e`) matches names **globally** with no area scoping, so this is a live
   risk the moment a world has two same-named places. **P0 fixes this.**

This is the groundwork for an evolving worldbuilding game (realms, kingdoms,
planets, alternate worlds), not just the current single-mansion chat RPG.

## The two axes (LOCKED)

Space and time are **orthogonal**. Do not conflate them.

| Axis | What it is | Mechanism | Used for | Carries into main story? |
| --- | --- | --- | --- | --- |
| **Space** | *Where* you are | Location graph + **world-roots** | Realms, planets, cities, buildings, rooms — incl. shadow/light realms, alien-abduction worlds | **Yes** — all on the main timeline |
| **Time** | *Which reality-thread* | **Timeline branches** ([TimelineBranchDoc](../../everlore-server/src/models/time.model.ts)) | Genuine what-ifs / dreams / rewinds that must NOT become canon | No — branches are sealed (a branch inherits its parent's past up to the fork; the parent never sees the branch) |

**Consequence:** canonical **world-drift is a space move on the main timeline**, NOT
a timeline branch. A three-realm fantasy = three **sibling world-roots**, all
`timeline_id: "main"`; crossing between them is movement; each realm accrues its own
canonical lore and memories carry forward. Reserve a timeline branch ONLY for a true
counterfactual ("what if I'd been abducted?" as a vision that doesn't count).

## Data model

Locations are ALREADY `type: 'location'` entities in the shared `entities`
collection (so they can be memory subjects/objects and `entity_edges` endpoints).
We add the **containment spine** and **world-root** identity:

```ts
LocationEntity extends EntityDoc {   // type: 'location'
  parent_id?: ObjectId | null        // immediate container (room → building → ...)
  world_root_id?: ObjectId | null    // top of the chain THIS place belongs to
  place_kind?: 'world' | 'region' | 'settlement' | 'building' | 'area' | 'room' | string
  // existing: canonical_name, name_normalized, aliases, location_state[],
  //           location_facts[], first/last_seen_sequence, mention_count, status
}
```

- **`parent_id`** is the spine: `dining room → mansion → Ashford → Veliscourt → Kingdom of Marr → (world root)`. A simple indexed back-pointer (NOT a stored full path) — re-parenting is then one cheap update and the subtree moves with it automatically.
- **`world_root_id`** is the denormalized top-of-chain, carried for O(1) scoping (so resolution/retrieval never has to walk the whole spine each turn). A place with no parent IS a world root (points at itself or null).
- **Non-containment relations** ("the neighbourhood OF the house", "Kingdom A allied with Kingdom B", "shadow realm is the mirror of the normal world") use the existing **`entity_edges`** with a relation `type` (e.g. `located_in` is implied by `parent_id`; `mirror_of`, `allied_with`, `borders` are edges). Containment = spine; everything else = edges.
- **Memory tagging is unchanged** — memories already copy the event's
  `location_anchor`. Once nodes are precise, place-recall is automatically correct.

### World roots

- A single-world game has **one implicit root** — we don't name it; the first place
  with no discovered parent is the root.
- A new root is minted ONLY when the fiction introduces a **distinct world/realm/
  plane** (shadow realm, alien world). Sibling roots are **unlimited**.
- `world_root_id` namespaces identity: `house@RealmA` and `house@RealmB` are
  different nodes because their roots differ → **no cross-world bleed, ever.**

## Identity & resolution rules (the duplication-proofing)

**Identity = canonical name + parent chain + world root.** Same name under a
different parent/root = a different place.

1. **Parent/area-scoped resolution (replaces global fuzzy).** A place name dedups
   only against **siblings under the active area / world-root**, never the whole
   instance. Returning to "the house" in Realm B resolves to B's house; it can never
   fuzzy-match A's. *(This is the core P0 change — today's resolver is global.)*
2. **Vague labels never mint.** A generic/relative label — "the room", "here",
   "inside", "outside", "this place" — does NOT create an entity. The server keeps
   the current cursor unless movement is narrated; on a move it attaches the new
   place under the right parent (see cartographer). "the room" during the dinner
   simply *is* the dining room. *(Location analog of "a bare descriptor is never a
   new person".)*
3. **Only a specific named place or a narrated move** can create or change a
   location.
4. Exact-name/alias match (within scope) first; bounded token-similarity second
   (within scope); else mint (within scope, under the right parent).

## Witness (LLM) vs Cartographer (server) — LOCKED

The model is a **witness**, never the mapmaker. It is unreliable at holding a whole
tree; it is reliable at the **local** observation. Same pattern as presence
carry-forward and travel detection (model emits easy deltas; server holds structure).

- **Extraction emits** (cheap fields in the pass that already runs — no extra call):
  - `current_place` — the specific place name in the scene ("the foyer").
  - `containment_hint` — the immediate container *only if the scene states it*
    ("inside the mansion"), else null.
  - `movement` — one of `none | deeper` (went *into* something) | `out` (left the
    current container) | `lateral` (same level, new place) | `world_shift` (crossed
    into a different realm/world).
- **Server (cartographer) decides** the actual graph edit from cursor + those hints:
  - `none` → stay on the current node.
  - `deeper` → new place's `parent_id` = current node.
  - `out` → new place's `parent_id` = current node's `parent_id` (one level up); if
    none known, the new place becomes a sibling under the same root.
  - `lateral` → same `parent_id` as the current node.
  - `world_shift` → mint/resolve a **world root**; new place hangs under it. On the
    **main timeline** (canon) unless the turn is explicitly a what-if (then fork).
- The witness/cartographer split is what makes `out` (leaving a container) correct —
  a pure cursor-inheritance rule (no model signal) cannot tell "into" from "out of".

## Lazy / emergent depth + partial ancestry — LOCKED

- Record only the levels the **fiction has actually revealed** (fog-of-war). Early
  game may be just `mansion → room`. "Neighbourhood" appears when you step outside;
  "city/kingdom" when the story names them. **Never pre-fill or invent ancestors**
  (inventing a kingdom name to fill a slot is the banned hallucination class).
- **Partial ancestry (`dining room → mansion → ?`) is correct and harmless:**
  - *Storyline:* the narrator only gets the **current** place + its facts/state
    ([context-packet.service.ts](../../everlore-server/src/services/context-packet.service.ts));
    retrieval keys off the node id + its *known* ancestors. An unknown parent just
    means "not yet grouped under a city" — no wrong behavior.
  - *UI:* renders to the depth known; the top known node is shown as a root until a
    parent is revealed.
- **Re-parenting on later reveal is cheap:** when "the mansion is in Veliscourt" is
  stated, set `mansion.parent_id = Veliscourt` (+ refresh `world_root_id` down the
  subtree). Children point at `mansion`, not at a path, so the whole subtree slots in
  with one update.
- Practical depth cap ~5–6 levels; no infinite nesting.

## Retrieval scoping (how carry-forward AND no-bleed coexist) — per arm

This already matches the code's multi-arm structure; we only add world-root identity.

- **Place-recall arm** ("what happened *here*?") → **scoped to the current node**
  (optionally its ancestors). Standing in the Light Realm, this arm pulls
  Light-Realm-here events only → **no wrong-place bleed.**
- **Semantic / keyword / entity-neighborhood arms** → **NOT place-restricted.** A
  canonical shadow-realm memory resurfaces in the main story when genuinely relevant
  ("remember the beast you slew in the Shadow Realm?") → **carry-forward preserved.**
- All arms remain **timeline-scoped** (`timelineEligibility`), so a sealed what-if
  branch never leaks into canon.

## Frontend — the location atlas

A nested **atlas** surface (sibling of the Bonds/codex surfaces in the Lore Tome):

- Tree: `world → region → settlement → building → area → room`, fog-of-war (only
  discovered nodes; undiscovered parents simply absent).
- Node detail: `location_facts` (canon), `location_state` (current condition),
  related locations (edges), who's been here, events/memories that happened here,
  "you are here" marker.
- Multiple world-roots render as **separate top-level realms**.
- "Go back to <place>" / fast-travel affordances key off the node id.

## Rollout

### P0 — vague-label guard — DONE (server only)
Stops fragmentation at the source. `isVagueLocationLabel` classifies generic/relative
labels ("the room", "here", "outside") matched as the WHOLE normalized label (so
"dining room" / "great room" stay specific); `resolveLocationAnchor` takes a
`viewpointMoved` param and returns null for a vague label on an UNMOVED turn (no match,
no mint → caller keeps the cursor), while a SPECIFIC place on an unmoved turn still
resolves (preserves the `a80bb10` "cursor follows on return" fix). Soft extractor nudge
added. `audit:location-resolution` extended (classification + guard scenarios), all green.

**Parent/area-scoped resolution moved to P1** — it needs the `world_root_id`/`parent_id`
spine to scope by, and the cross-world bleed it guards against can't occur until
multi-root worlds exist (P1). Shipping it in P0 would be untestable dead code.

### P1 — the spine (data model + cartographer + scoped resolution) — DONE (`f95099d`)
- Added `parent_id`, `world_root_id`, `place_kind` to location entities. Unique index
  now `idx_entities_instance_type_root_name` (includes `world_root_id`) so same-named
  places coexist across realms; legacy/non-location/single-world rows keep
  `world_root_id` null (constraint reduces to the old key). Added `instance+parent_id`.
- **Parent/world-root-scoped resolution** (the moved P0(b)): `resolveLocationAnchor`
  exact + fuzzy now scoped to one world-root — closes the cross-realm bleed.
- Witness fields `containment_hint` + `movement` (none/deeper/out/lateral/world_shift);
  server cartographer `placeLocation` does the placement + re-parent reveal + world-root
  minting (self-rooted, main timeline). Audit + dev-instance backfill done.
- ~~KNOWN LIMIT: subtree `world_root_id` refresh on a cross-root re-parent not yet
  done~~ — **RESOLVED (server `c3b04f0`, June 12 2026).** `refreshSubtreeWorldRoot`
  reroots a re-parented node + its descendants (self-rooted nested worlds stop the
  descent); the re-parent reveal now calls it so the denormalized root stays
  consistent with the spine by construction. Deterministic + audited; no-op in the
  single-world dev instance.
- Extraction emits `current_place` / `containment_hint` / `movement`.
- Server cartographer logic (attach under correct parent; handle deeper/out/lateral/
  world_shift); world-root minting on `world_shift` (main timeline).
- Re-parenting on later reveal (+ subtree `world_root_id` refresh).
- **Backfill** the current instance's 8 locations under a `mansion` building +
  `exterior` area, one implicit world root.
- Audits: cartographer in/out/lateral/world_shift scenarios; re-parent correctness;
  rewind-safety of the spine.

### P2 — the atlas UI — DONE (server `e633956` + app `c8d7bbc`)
- `listLocations` returns the full spine (every location entity incl. empty
  containers, with parent_id/world_root_id/place_kind + counts). The Places tab is a
  fog-of-war nested tree (place-kind glyphs, current place highlighted + auto-expanded,
  fold/unfold, tap → journal). Hardened `merge:location` to re-point the memory PLACE
  anchor (was stranding merged-away memories on a deleted entity → ghost atlas node);
  added `repair-orphan-place-anchors.ts`.
- DEFERRED to P2.5/P3: non-containment relation edges (`mirror_of`, `allied_with`,
  `borders`, …) — nothing mints them yet (needs an extractor pass), so surfacing would
  show nothing today.

### P2.6 — resolution accuracy / movement hardening — DONE
The witness/cartographer split only works if the witness reliably reports movement
and place names — and the small model does NOT. "I go to my room and shut the door"
came back with `viewpoint_moved:false` and a vague/cursor-equal place, so the cursor
stuck on the dining room and presence never reset (parents bled into the bedroom).
**Location precision IS place-recall precision** (see "Why"), so this hardening goes
BEFORE P2.5's relation-edge cosmetics. Fuzzy matching was not at fault — the fix is
at the witness seam with deterministic server math, the same pattern as the F3
presence fold and the codex name-grounding backstop:
- `worker/lib/movement-signal.ts` — `detectNarratedMovement` (locomotion in the
  player's own narrated action, the most reliable move signal) + `resolvePossessiveRoomName`
  ("my room" → "<owner>'s room", first-person only). Pure + `audit:movement` (25/25).
- `generation.processor` — owner-scoped name override when the player retreats to
  their own space and the model whiffed (placed lateral); `viewpoint_moved`
  corroborated from the narrated move ONLY when the resolved name is a real different
  place (an ambiguous "going to" on a stay-put turn can't reset the scene); presence
  resets whenever the resolved location ENTITY changes (deterministic). Location +
  `present_characters` are thus fixed by one coupled change.
- Live instance repaired (`repair-bedroom-anchor.ts`). KNOWN GAP: no live generated
  turn yet (seq 27+).

### P3 — multi-world maturity — DEFERRED (content-gated)
- Multiple world-roots in anger; "go back across realms"; per-realm lore at scale.
- Timeline-branch what-ifs layered on top (optional per scenario) — e.g. a vision /
  dream realm that must not rejoin canon.
- **Status (June 12 2026):** the SPINE is complete and root-consistent — world-roots,
  world-root/area-scoped resolution + the unique index, `world_shift` minting, and the
  subtree `world_root_id` refresh. P3 is now an *exercise-when-content-arrives* item,
  not unbuilt infrastructure: the first time a world introduces a second realm, drive
  it through the existing cartographer and verify, rather than building speculative
  multi-world machinery ahead of any realm.

### P2.5 — non-containment relation edges — DEFERRED (content-gated)
`mirror_of` / `allied_with` / `borders` need an extractor pass to MINT them, and the
single-world content produces none — building the producer + atlas surface now shows
nothing and can't be live-verified (same call as Phase 4 BM25). Reopen when content
introduces cross-place relations. **Trigger prompt:** _"Everlore content now needs a
non-containment place relation (e.g. allied kingdoms / a mirror realm / bordering
regions). Read `LOCATION_GRAPH.md` — this is the deferred P2.5 item. Add the witness
field + cartographer edge mint on `entity_edges`, surface it in the atlas node detail,
then verify per `LIVE_VERIFICATION.md`."_

### Location Graph — close-out (June 12 2026)
Everything buildable AND verifiable without new content is DONE: P0–P2.6, the
intra-world same-name collision fix, and the subtree `world_root_id` refresh. The
remaining items (P2.5 relation edges, P3 multi-world, open-world limits #2–4) are all
**content-gated** — each has a concrete reopen trigger above / in "Open-world limits".
The stack degrades gracefully on every one of them, so they are pulled in when content
demands, not ahead.

## Locked decisions

- Space vs time are orthogonal; **world-drift is canonical space travel on the main
  timeline**; timeline branches reserved for true what-ifs.
- **Lazy/emergent depth**; never invent ancestors; partial ancestry is valid.
- **Witness/cartographer**: LLM emits local place + movement hints; server owns the
  map.
- **Parent/world-root-scoped identity & resolution**; vague labels never mint.
- Containment = `parent_id` spine; other relations = `entity_edges`.

## Open questions (resolve at P1 kickoff)

- Default starting depth granularity for a brand-new world (building→area→world, vs
  deeper) — start shallow, grow.
- Exact `place_kind` taxonomy (free string vs fixed enum).
- Whether the place-recall arm includes ancestor nodes (e.g. mansion-level memories
  when standing in a room) or strictly the current node.

## Open-world limits (known, deferred — documented June 2026)

The witness/cartographer + movement-signal stack is sufficient for the **current
loop**: a single protagonist, one active scene at a time, sequential named travel.
Stress-testing it against genuine open-world play surfaced gaps that are **logged,
not yet built** — pull each in when the *content* demands it. Ordered by severity.

1. **Intra-world same-name collision — FIXED (area-scoped, June 2026).** Was a latent
   bug: P1 scoped resolution to `world_root_id` only, AND the unique index was
   `(instance,type,world_root_id,name)` — so within one world a name was unique at the
   DB level, a 2nd `"tavern"` in another town couldn't even mint (insert failed → FUSED
   onto the first → place-recall bled Town-A memories into Town B). Fixed in two coupled
   parts: (a) the unique index gained `parent_id`
   (`idx_entities_instance_type_root_parent_name`, old one deprecated/dropped at
   startup — looser key, no migration violation); (b) `resolveLocationAnchor` gained
   `areaScope` — a name match counts only if the candidate shares the intended
   placement's **AREA** (its nearest world/region/settlement/building ancestor, via
   `resolveAreaId`). So `"tavern"@Ashford` ≠ `"tavern"@Riverton`, while `dining room`
   and `hallway` in one mansion still resolve (no regression to returns) — and area
   (not immediate parent) scoping deliberately absorbs movement mislabels within a
   building. The cartographer passes the intended parent's area; with no area anchor it
   falls back to the lenient world-root match. Inert for the single-building dev world.
   `audit:location-resolution` covers collision/reuse/deeper-safety. KNOWN RESIDUAL: a
   same-name place introduced with NO container signal at all still uses the lenient
   match (genuinely ambiguous). Multi-settlement content is now safe to ship.

2. **Traveling-party presence (MISSING MODEL — needs a feature, not a patch).** Presence
   is a flat "who is in THIS scene", rebuilt on every move (`sceneBroke` resets it —
   `viewpointMoved`, a time skip, or `placeEntityChanged`). That is correct for "I left
   the dinner, my parents don't follow", but WRONG for "my companions ride to the
   capital WITH me" — companions drop unless the arrival prose re-names them every turn
   (the F3 "quiet character flickers to elsewhere" failure, but across a move). Open
   world needs TWO presence sets: **co-located locals** (reset on move) vs **party /
   travelling-with** (persist across moves until an explicit part/farewell). Build:
   a persistent `travelling_with` set on the instance, seeded by explicit join signals
   and cleared by explicit departures, unioned into presence after a move so the party
   survives a scene break while locals still reset. Defer until companions are real
   gameplay.

3. **Dedup at world scale (DEGRADES, not broken).** The KNOWN PLACES roster fed to the
   witness is capped at 30 (recency/mention-sorted) and the fuzzy pass only sees bounded
   candidates. A world with hundreds of places means an old, long-unvisited place may be
   off the roster → the model can mint a variant the fuzzy pass might miss. Exact returns
   are always fine (indexed). Revisit with a smarter hot-set (spatial/recent-neighbourhood
   bias) only if duplicate long-tail places actually appear at scale.

4. **Exotic, unmodeled (lowest priority).** (a) **Multi-hop montage** ("three weeks
   crossing the mountains to the capital") mints only the END place; intermediate places
   never exist (fog-of-war — acceptable, but no journey place-recall). (b) **Mobile
   containers** — a ship/airship that is itself a moving "place" (stable container,
   changing world-position) has no representation. (c) **Parallel scenes / party split** —
   a single instance cursor + single present-roster assume ONE active scene; a true
   "meanwhile, elsewhere" split isn't modeled (side-chats cover only a single alt-thread).
   Pull in per scenario if/when a world needs it.

**Takeaway:** the movement-signal/witness/cartographer split is the right foundation and
degrades GRACEFULLY (worst case: a sticky cursor or a flickered companion — never silent
identity corruption) EXCEPT for limit #1, which can corrupt by fusing distinct places.
#1 is the one to fix proactively; the rest are content-driven.

## Repair tooling (exists)

- `bun run merge:location <iid> "<keep>" "<dupe>" [...]` — folds duplicate location
  entities (aliases/mentions/facts/state), re-points memory + edge refs, rewrites
  stored event `location_anchor`s + travel labels, repoints the instance cursor,
  deletes the dupe. Used to repair the current instance (12 → 8 locations). The
  sibling of `merge:character`.
