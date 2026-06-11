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

### P1 — the spine (data model + cartographer + scoped resolution)
- Add `parent_id`, `world_root_id`, `place_kind` to location entities (+ indexes).
- **Parent/world-root-scoped resolution** (moved from P0): scope exact + fuzzy matching
  in `resolveLocationAnchor` to siblings under the active world-root instead of the
  global instance set — closes the cross-realm "same-named house" bleed. Today's
  resolver (`a12e94e`) is GLOBAL; this is exercisable once roots exist.
- Extraction emits `current_place` / `containment_hint` / `movement`.
- Server cartographer logic (attach under correct parent; handle deeper/out/lateral/
  world_shift); world-root minting on `world_shift` (main timeline).
- Re-parenting on later reveal (+ subtree `world_root_id` refresh).
- **Backfill** the current instance's 8 locations under a `mansion` building +
  `exterior` area, one implicit world root.
- Audits: cartographer in/out/lateral/world_shift scenarios; re-parent correctness;
  rewind-safety of the spine.

### P2 — the atlas UI + relations
- Nested atlas screen; node detail; world-root grouping.
- Non-containment relation edges (`mirror_of`, `allied_with`, `borders`, …) +
  surfacing in the prompt's place context.

### P3 — multi-world maturity
- Multiple world-roots in anger; "go back across realms"; per-realm lore at scale.
- Timeline-branch what-ifs layered on top (optional per scenario) — e.g. a vision /
  dream realm that must not rejoin canon.

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

## Repair tooling (exists)

- `bun run merge:location <iid> "<keep>" "<dupe>" [...]` — folds duplicate location
  entities (aliases/mentions/facts/state), re-points memory + edge refs, rewrites
  stored event `location_anchor`s + travel labels, repoints the instance cursor,
  deletes the dupe. Used to repair the current instance (12 → 8 locations). The
  sibling of `merge:character`.
