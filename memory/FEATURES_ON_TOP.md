# Features Enabled By Infinite Memory

Once the memory graph exists, Everlore can support features that would otherwise
require separate bespoke systems. The graph becomes the shared substrate.

## Direct Side-Character Chats

Players can open private chats with side characters while preserving global
canon.

Memory requirements:

- side chat events use the same event ledger
- active character is pinned in the retrieval packet
- relationship memories are updated from the private chat
- secrets can be scoped to characters who know them
- main world narration can later reference what happened privately if the
  relevant characters know it

Example:

```text
The player privately tells Mira about the prophecy.
Mira now knows the secret.
Arlen does not.
The next group scene should not let Arlen act on that knowledge.
```

## Travel And Location Gameplay

Enabled features:

- current location state
- location history
- travel time
- recurring emotional associations
- city-specific NPCs
- place-based quests
- "what happened here before?" retrieval

## Calendar UI

The player can inspect events by story date:

- normal calendar
- magical calendar
- moon phases
- eras
- festivals
- time skips
- flashbacks
- altered timelines

Calendar entries should link back to source events, summaries, memories, and
location state.

## Relationship Ledger

Relationships become inspectable and playable:

- trust
- affection
- fear
- rivalry
- debt
- secrecy
- betrayal
- forgiveness
- unresolved tension

The model can use the ledger to write emotionally consistent scenes.

## Quest And Promise Tracking

Promises are a memory type:

```text
The player promised Mira they would return before the red moon.
```

The graph can track who made the promise, who heard it, deadline/story date,
fulfilled/broken/current status, and emotional consequence.

## Living Locations

Locations can change over time:

```text
The market burned down.
The player funded its rebuilding.
The rebuilt market now has a statue honoring the player.
```

The world can remember how the player changed a place thousands of turns later.

## Memory-Aware Recaps

The game can generate:

- "Previously on..."
- character-specific recaps
- relationship recaps
- city history
- timeline recaps
- "What Mira remembers about you"

These recaps come from graph/context projections, not raw transcript dumps.

## Time-Travel Stories

The same architecture supports loops, alternate branches, paradoxes, erased
timelines, characters with different subjective memories, and future knowledge.

The key is timeline-aware memory retrieval. A character should only remember
what their timeline and subjective experience allow.

## Player Memory Controls

Players can inspect and edit:

- memories
- character cards
- relationship facts
- timeline events
- location facts

Every edit should trigger projection repair.

## Search And Discovery

Chronicle can evolve into a powerful memory browser:

- search by character
- search by place
- search by promise
- filter by timeline
- jump to date
- show all events affecting a relationship
- show all memories sourced from a scene

This keeps exact history player-visible without making Play load everything.

