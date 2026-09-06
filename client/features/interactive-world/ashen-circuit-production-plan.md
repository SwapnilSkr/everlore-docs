# Ashen Circuit Production Plan

## The promise

Ashen Circuit is a beautiful, interactive medieval story world where the player
explores a connected map, meets characters with durable context, makes actions
in illustrated scenes, and gradually opens new places and quest paths. It is
not a chat UI with a decorative map, and it is not a full 2D action fighter.

The first release is a dense, memorable district. It earns the feeling of an
open world through connected places, changing states, character schedules,
recurring consequences, and secrets—not raw map size.

## What we build first

```text
Ashen Circuit / Chapter I: The Cinder Oath

Cinder Arena (start)
  ├─ Ember Market        [unlocks after first arena scene]
  ├─ Fighters' Barracks  [unlocks through Cassian's trust]
  └─ Ash Gate            [unlocks through the sealed-letter quest]
       └─ Cinder Underway [later chapter]
```

Each location has a unique illustrated scene, a small interaction set, active
NPCs, discoverable state, exits, and at least one reason to return. The first
chapter needs **four excellent places**, not one hundred thin ones.

## The player loop

```text
World map → choose route → travel transition → illustrated location scene
  → inspect / approach / speak / use an action / leave
  → resolved world event → quest + memory + relationship changes
  → map, characters, and available choices update
```

Chat opens from a scene and receives its context. It is never the only way to
perform basic world actions. The screen should always offer a compact set of
legible actions plus a chat affordance for unusual intent.

## Visual system

### Scene composition

Every scene follows one predictable, beautiful stack:

1. environment background (wide, no UI baked in);
2. optional midground / foreground overlays for depth;
3. character cutouts at named scene marks;
4. atmospheric effects: particles, smoke, fire, rain, light sweep;
5. narrative panel, choices, quest / relationship state.

The camera does not simulate a 2D game world. It pans, subtly zooms, fades,
and shifts focus between scene marks. This makes hand-authored art feel alive
without requiring an animation for every movement.

### World-map system — landmark scene graph

The map is a composed world scene, not an image with pins or preview cards.
Each map is built from a terrain layer plus independently addressable landmarks.
The arena, gate, market, barracks, bridges, and entrances are real map entities
with their own art layers, clickable footprint, state, and motion. Tapping the
drawn Coliseum selects the Coliseum itself; its torches, banner, crowd glow, or
lock seal can react before the travel affordance appears.

```text
MapScene
  terrain base (non-interactive land, water, roads, relief)
  landmark entities (transparent art, z-order, hit polygon, state variants)
  route mesh (authored world-coordinate paths)
  atmosphere (embers, smoke, water, light)
  camera (pan, pinch zoom, focus, travel)
```

The renderer is reusable; the content is genre-specific. Medieval uses
painted terrain and illustrated landmarks; cyberpunk uses city blocks, transit
lines, signage and rain; football uses streets, venues, fixtures and crowds.
No generic location card is a map entity.

**Landmark interaction contract**

Each landmark declares a normalised hit polygon, world-space anchor, z-index,
visual layers, idle motion, state variants, and named actions. Its visual state
is derived from server-owned quest flags. The hit polygon is used for both tap
testing and selected-outline rendering—there is no unrelated rectangular UI
target over a map.

### Motion vocabulary

Use a restrained shared vocabulary: arrival slide, breathing drift, cloth sway,
blink, lean-in, portrait focus, object glow, recoil/shake, tint flash, particle
burst, and exit fade. A motion is triggered by a world event and lasts less
than a second except idle ambience. Sound and haptics carry much of the impact.

### Asset contract

```text
assets/art/ashen_circuit/
  maps/cinder_ward/terrain_vN.png
  maps/cinder_ward/landmarks/<location>/base_vN.png
  maps/cinder_ward/landmarks/<location>/effect_vN.png
  maps/cinder_ward/landmarks/<location>/locked_vN.png
  locations/<location>/<scene>_midground_vN.png
  locations/<location>/<scene>_foreground_vN.png
  characters/<character>/<pose>_vN.png
  characters/<character>/<expression>_vN.png
  effects/<family>/<effect>_vN.png
  maps/cinder_ward_vN.png
```

Generate or commission assets in families. Maintain a visual bible: camera
angle, horizon line, scene-mark positions, lighting direction, palette, scale,
depth order, export dimensions, and no-text/no-UI rule. Characters require a
base neutral cutout first; poses and expressions are additive, not prerequisites
for adding the character to the world.

## World data and ownership

The source of truth belongs on the server once the local vertical slice is
validated. The client is a fast renderer and local predictor.

```text
WorldDefinition: locations, routes, entry rules, visual references
LocationState: unlocked, visited, time / weather, active scene, discovered facts
NpcState: location, schedule, relationship, goals, memory references
QuestState: stage, prerequisites, flags, rewards, branch locks
WorldAction: move | inspect | approach | speak | choose | use | leave
WorldEvent: actor, action, targets, outcome, state changes, memory candidates
```

All visible changes must be derivable from a `WorldEvent`. This gives us saves,
cross-device resume, testing, debugging, and a reliable context packet for the
chat / agent system.

## NPC and memory model

Each important NPC has identity, visual family, location / schedule, goals,
relationship state, secrets, and memory links. For any scene or chat turn,
compose a small context packet from stable facts, current scene, quest stage,
recent witnessed events, and retrieved relevant memories. Do not ask a model to
infer canonical state from dialogue history.

## Quest design

Quests are authored state machines with optional branches, not unconstrained
LLM stories. AI can create scene texture, barks, rumours, and bounded optional
encounters. It cannot invent a route, canon reward, or completion state.

The Cinder Oath opening quest:

1. Witness the arena summons.
2. Choose how to respond to Cassian or the sealed letter.
3. Unlock a distinct lead in the Market or Barracks.
4. Learn a fact that changes an Ash Gate encounter.
5. Make a choice that changes Cassian's relationship and a later scene.

## Implementation order

1. **World shell (now):** local map, location lock rules, illustrated scene,
   scene marks, choice panel, transition language.
2. **First asset kit:** Cinder Arena, Ember Market, Cassian neutral + two
   expressions, one market NPC, map artwork, shared smoke/fire/ember effects.
3. **Local state graph:** typed routes, action reducer, event log, quest flags,
   relationship values, save / restore.
4. **Connected chapter:** four locations, Cinder Oath, three NPC contexts,
   two meaningful unlock paths.
5. **Server integration:** canonical actions/events, persistence, context
   packets, chat handoff, analytics and content tooling.
6. **Scale only after validation:** extract reusable scene, map, quest, and
   asset-kit contracts for the next medieval world.

Backend, database, and S3 asset-manifest details live in
[Interactive Worlds: Backend, Database, and Assets](../../../server/INTERACTIVE_WORLDS.md).

The Flutter transport boundary is
`lib/features/interactive/data/interactive_world_repository.dart`. The first
screen supports two modes: no `instanceId` is the deliberate local authoring
preview; `?instanceId=<world-instance-id>` loads canonical state from the API
and sends typed actions through this repository. The visual map renderer does
not need to change between them. The next navigation task is passing the
currently selected realm/play-instance id to this route automatically.

That handoff is now available from the Ashen Circuit Realm screen through
**Enter the Cinder Ward**, which preserves its `instanceId` in navigation. The
same persistent state can therefore be opened from a player’s Realm instead of
only through the authoring-preview link.

**Progress:** Cinder Arena, Ember Market, Fighters’ Barracks, the Cinder Ward
map, Cassian Vale, and Nara Voss are now versioned source assets. Barracks is
an authored playable branch: Cassian’s trust opens it, the seal action opens
Ash Gate, and the consequence is visible on the same graph. Ash Gate is now a
versioned scene too; offering its letter opens the Cinder Underway. The next
art pass adds the Underway, character expression overlays, and atmospheric
effect layers.

**Map progress:** Cinder Ward uses a high-fidelity authored medieval atlas as
its terrain skin—not a background that contains playable pins. `map_style`
selects the skin, while server-owned location coordinates drive the actual
interactive place cards, travel routes, locks, selection, and travel behavior.
It supports pinch pan/zoom and dedicated zoom/reset controls. The atlas remains
immutable in S3; the dynamic mesh is the renderer above it, not painted into it.

**Superseded:** the atlas-plus-location-card prototype is being removed. It did
not meet the landmark-scene-graph contract. Its local and S3 map artifacts are
retired before the new terrain and transparent landmark layers are published.

**State progress:** `move` and `witness_arena` now resolve server-side, write
the per-instance interactive state, and append an `interactive_world` entry to
Everlore’s canonical event ledger. The first arena witness, Cassian trust, and
Nara meeting each become one importance-7 unresolved memory atom for future
agent context; repeated visits do not duplicate them. The Barracks seal study
is now included in that idempotent promotion policy.

## Success criteria for Chapter I

- A new player understands where they are, what can be done, and why a locked
  place is locked.
- At least one choice visibly changes a later location, person, or route.
- Returning to a location can show new people, dialogue, or state.
- The player can chat with an NPC *from* a meaningful scene context.
- The visual experience feels premium on a phone with no full-frame character
  animation required.
