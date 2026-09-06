# Ashen Circuit: Interactive World Direction

**Build plan:** [Ashen Circuit Production Plan](./ashen-circuit-production-plan.md)

Ashen Circuit is Everlore's first **action-first interactive world**. It does not replace the chat RPG: chat, realms, character memory, and open-ended role play remain available. The interactive layer turns selected moments into authored, playable scenes where movement, timing, combat, sport, investigation, and traversal decide what happens.

The target feeling is an open world with understandable rules, not an AI chat wrapped in a map. Players should be able to roam, see who and what is nearby, choose an action, and receive a consequence that the world and its characters remember.

> **September 2026 visual direction:** Do not build a full 2D fighting engine
> for this. The production model is an illustrated, state-driven scene: rich
> location art, high-quality character cutouts, clear action choices, and tiny
> reusable motion / effect cues. The earlier frame-sheet and free-rig work are
> research prototypes, not the visual path to ship.

## Visual interaction direction: illustrated scenes, not 2D combat

The King’s Choice reference is useful because a battle is read through a rich
location illustration, two large character portraits, concise state UI, and
selective feedback—not a fully animated side-scrolling fight. Everlore should
borrow that presentation economy, not its numbers-first battle loop.

```text
Scene = authored location art + foreground character cutouts + state UI

arrival    → character eases into its scene mark
idle       → breathing, cloth drift, blink, light parallax
attention  → small lean, portrait focus, dialogue bubble
choice     → card / object highlight, camera nudge, sound cue
action     → brief lunge or shake, weapon / hand overlay, VFX
impact     → target recoil, tint flash, number / state change
departure  → fade or walk-out transition to the next authored scene
```

Motion is a cue, not an animation-heavy game. A character needs one strong
transparent portrait plus a small optional overlay kit: blink, arm / weapon,
cloak, reaction, and effect layers. Those layers are shared by visual family
where possible. Major characters earn a bespoke pose or two; supporting
characters use the family kit, palette, equipment, expression, and context to
stay distinct.

Exploration should be an authored location graph with map nodes, routes, and
scene transitions—not free pixel-level walking. The player chooses a visible
route or point of interest, sees a short travel transition, and arrives in a
new scene. It remains open because routes unlock, people move, places change,
and choices alter what can appear there.

Meeting an agent NPC should establish their place and presence *before* chat
opens. The player can inspect, approach, speak, offer, threaten, recruit,
leave, or use a world-specific action. Chat gets the exact location, scene
state, relationship facts, and witnessed events as context.

## Product shape

```text
Template (Medieval)
  └─ World (Ashen Circuit)
       ├─ Location graph (arena, market, barracks, undercity...)
       ├─ Character cast and their individual context
       ├─ Quest / event graph
       ├─ Interactive scenes (combat, travel, sport, stealth, puzzles)
       └─ Chat / role-play layer for free conversation and exceptional intent
```

The map is deliberately authored. A world can feel large through a dense graph of meaningful places, time-gated routes, recurring NPC schedules, and changing states; it does not need one hundred bespoke real-time environments before the first release. Reusable location kits establish the base, while landmark locations, major characters, and pivotal quest moments receive bespoke art.

## Design rules

1. **Actions are first-class events.** A sword swing, choosing a route, winning a bout, bribing a guard, or helping an NPC must create a structured event, not only a line of prose.
2. **Chat is a capability, not the game loop.** Use it for conversation, negotiation, free expression, and edge cases. Do not require a player to type every move in a fight or every step of travel.
3. **Simulation owns truth.** The client renders and predicts; the encounter / world resolver validates outcomes and emits canonical events.
4. **Author constraints; generate variation.** AI may write barks, optional encounters, descriptions, and permutations within a state machine. It must not invent canon, rewards, traversal, or combat results.
5. **Every reusable system has an authored escape hatch.** A generic arena, motion rig, or location kit is efficient. Signature bosses, landmarks, and scenes can override it.

## The current vertical slice

Ashen Circuit starts as one medieval combat-story world, not a platform for 20 worlds on day one. Its proof slice is:

- one arena location with movement, range, sword, and shield decisions;
- Cassian Vale and one rival using the same base fighter definition but different skins, weapons, shields, and combat profiles;
- one short quest chain whose outcome writes memory and changes a later scene;
- travel to two or three adjacent locations; and
- chat available from the character / encounter context.

This proves the expensive parts early: action-to-consequence flow, character context, content authoring, reusable art, and replayable encounter logic.

## Runtime model

```text
Player input
  → typed action command
  → encounter / world resolver
  → canonical world event
  ├─ client animation and UI feedback
  ├─ quest progression and rewards
  ├─ character-memory write candidates
  └─ chat context update
```

Examples of typed commands: `move(locationPoint)`, `attack(light)`, `guard(begin/end)`, `talk(npcId)`, `interact(objectId)`, and `travel(routeId)`. Each has preconditions, costs, deterministic resolution where appropriate, and an event payload. The same event format lets a player resume across devices, lets a chat scene reference what happened, and keeps the game testable.

### Character context and memory

An NPC does not need the complete history in its prompt every time. At runtime assemble a bounded context from:

- stable identity and relationship facts;
- current place, encounter, and quest state;
- recent witnessed events;
- a small number of retrieved, relevant long-term memories; and
- author-owned goals, secrets, and dialogue constraints.

The memory system records structured facts and event summaries first. Model generated prose is a presentation of that state, never the sole record of it. See the existing [memory architecture](../../../memory/MEMORY_ARCHITECTURE.md) and [server world model](../../../server/WORLD_MODEL.md).

## Animation and character scale

Do **not** generate a full frame sequence for every character/action. That creates huge asset cost, visual drift, and repetitive movement.

Ashen Circuit's free path does not require Spine or Rive licensing:

```text
Rig family
  ├─ shared skeleton / motion rules (gladiator_light)
  ├─ shared action set (idle, locomotion, guard, attacks, hit, death)
  └─ shared VFX / timing rules

Character definition
  ├─ skin / layered cutouts
  ├─ weapon and shield attachments
  ├─ palette and silhouette modifiers
  ├─ combat profile and timing offsets
  └─ authored barks, goals, quest role, memories
```

The Flutter proof uses a native Flame modular rig: torso, legs, weapon arm, shield arm, sword, and shield are posed independently. It intentionally uses painted placeholder pieces, so it proves interaction and animation ownership without pretending that it is final art. Production art replaces those pieces with aligned transparent cutouts from a consistent character kit. The state and input contracts remain unchanged.

For a quality bar, commission or generate **layered source art**, not a finished flat illustration: head, torso, upper/lower arms, upper/lower legs, cloak, weapon, shield, and optional hair / accessory layers. Standardise pivot points, canvas size, facing, lighting, and palette per rig family. Keep a small library of authored motion curves, then use different profiles, timing offsets, weapon arcs, hit reactions, spacing, and encounter AI to create variety.

### Preventing repetitive combat

The rig maps semantic actions to a list of compatible clips. A deterministic selector excludes the most recent clips before selecting the next one. The seed comes from the encounter / event sequence so it is replayable and can be validated server-side. Variety should also come from distance, enemy intent, stamina, terrain, posture, and interruptions—not only from random animations.

The current client contracts are in:

- `lib/features/interactive/domain/character_rig.dart`
- `lib/features/interactive/data/ashen_circuit_roster.dart`
- `lib/features/interactive/game/combat_action_selector.dart`
- `lib/features/interactive/game/ashen_circuit_game.dart`

## Content production plan

Build one reusable medieval kit before multiplying worlds:

1. **World grammar:** location types, route gates, event schemas, quest beats, NPC roles, and encounter objectives.
2. **One rig family:** a gladiator / adventurer kit with skin, equipment, and silhouette variation; its motion actions and VFX language.
3. **One location kit:** arena, street, interior, and dungeon variants built from common foreground / midground / background layers.
4. **One authored world:** Ashen Circuit's signature places, cast, and three short quest arcs.
5. **Extraction:** turn only proven repetition into template configuration.

New template genres (cyberpunk, isekai, football) should share world-event, memory, quest, and interaction contracts while owning their own visual kits and mechanics. Football is not medieval combat with a different skin; it needs its own skill checks, spatial rules, encounter goals, and motion family.

## Status and next work

Implemented locally in the Flutter app and verified on an Android emulator (Pixel-class, API 37) at `/interactive/ashen-circuit/lab`: an entry from Worlds, a Flame arena preview, tap-to-move with clamped arena bounds, one-shot sword attacks, hold-to-guard that actually blocks a telegraphed rival commit, the reusable character / rig contract, deterministic no-immediate-repeat attack selection, and a loadout panel rendered straight from `CharacterDefinition` / `CombatProfile`.

Two rules the first pass got wrong and the on-device pass caught, worth keeping when the painted parts are replaced by art:

- **The shield is the leading arm.** Both fighters originally held the shield on the side away from their opponent, so a raised guard read as decoration. The weapon is the trailing arm; the shield leads.
- **Reach and spacing are expressed in rig widths, not pixels.** A fixed pixel reach let the two rigs stand inside each other at strike range, which hid the swing behind the other fighter's torso. Depth is resolved by sorting on ground-line Y.

To boot straight into it while iterating: `flutter run --dart-define=EVERLORE_START_ROUTE=/interactive/ashen-circuit/lab`.

Next implementation milestones:

1. Replace painted placeholders with the first aligned, layered Gladiator Light PNG kit and preserve the same component hierarchy.
2. Define the shared `WorldAction` / `WorldEvent` client-server schema and persist an encounter result.
3. Add location graph traversal and a first quest gate that responds to the arena outcome.
4. Feed compact encounter events into NPC context / memory retrieval.
5. Add authored sound, impact VFX, accessibility settings, and device performance budgets before expanding the cast.

## Explicit non-goals for the prototype

- an unbounded generated map;
- a hundred places or two hundred unique characters before the vertical slice;
- AI-generated combat outcomes outside deterministic rules; and
- treating a generated image sheet as a scalable animation system.
