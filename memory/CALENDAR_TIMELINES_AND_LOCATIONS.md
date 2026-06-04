# Calendar, Timelines, And Locations

Long-running worlds need time and place as first-class memory concepts. A player
may spend years in one world, travel between cities, talk to side characters in
private scenes, enter flashbacks, or alter history through time travel.

## Three Kinds Of Time

| Time | Meaning |
|------|---------|
| Real time | When the server stored the event |
| Sequence time | When the player experienced the event |
| Story time | When the event happened inside the world |

Example:

```text
Real time: June 4, 2026
Sequence: turn 8,421
Story time: Year 342, Frostwane 12
Event label: the night before the coronation
```

If the player later enters a flashback to childhood:

```text
Real time: June 4, 2026
Sequence: turn 8,700
Story time: Year 318, Highsun 3
Timeline: main
```

Sequence increased, but story time moved backward.

## TimeAnchor

Use a reusable time anchor object:

```ts
TimeAnchor {
  real_time: Date
  sequence: number
  story_calendar?: {
    calendar_id: string
    year?: number
    month?: number
    day?: number
    era?: string
    label?: string
  }
  event_time_label?: string
  timeline_id: string
  causal_parent_event_ids: ObjectId[]
  subjective_entity_times?: Record<string, string>
}
```

## Magical Calendars

Worlds may define custom calendars:

```ts
WorldCalendar {
  id: string
  template_id: ObjectId
  name: string
  eras: string[]
  months: Array<{ name: string; days: number }>
  weekdays?: string[]
  moons?: Array<{ name: string; cycle_days: number }>
  season_model?: Record<string, unknown>
}
```

The memory architecture should not assume Earth calendars. It should store both
structured fields and human-readable labels.

## Time Travel

Time travel requires timeline branches:

```ts
TimelineBranch {
  id: string
  instance_id: ObjectId
  name: string
  parent_timeline_id?: string
  forked_at_sequence: number
  forked_at_story_time?: TimeAnchor
  status: "active" | "collapsed" | "alternate" | "erased"
}
```

Rules:

1. Sequence order remains the player's experienced order.
2. Story time may move forward, backward, or sideways.
3. Timeline ID tells retrieval which version of reality applies.
4. Causal edges explain why one event matters to another.
5. Memories from erased/collapsed timelines may remain as dreams, scars,
   paradoxes, or forbidden knowledge depending on world rules.

## Locations

Locations should be graph entities, not plain text strings.

```ts
LocationEntity {
  id: ObjectId
  instance_id: ObjectId
  canonical_name: string
  aliases: string[]
  type: "city" | "room" | "kingdom" | "forest" | "planet" | string
  parent_location_id?: ObjectId
  current_state: string[]
  permanent_facts: string[]
  first_seen_sequence: number
  last_seen_sequence: number
}
```

Location memories should capture:

- what happened here
- who was present
- what changed physically
- what changed politically
- what emotional associations the player has with the place

Example:

```text
The Ash Bridge is where the player betrayed Mira, and even after reconciliation
Mira becomes quiet when they return there.
```

That is both a location memory and a relationship memory.

## Travel

Travel should create events and update location state:

```text
Player traveled from Ashen City to Moon Temple.
Travel took three in-world days.
Mira became distant during the journey.
Bandits destroyed the western road behind them.
```

These should project into:

- current location
- calendar advancement
- location graph
- relationship memories
- possible quest/faction state

## Retrieval With Time And Place

The context packet should answer:

```text
Where is the player now?
What happened here before?
Which characters are nearby?
What date is it in this world?
Which timeline are we in?
Are there unresolved memories tied to this place/time?
```

This makes travel and calendars meaningful to gameplay rather than cosmetic.

