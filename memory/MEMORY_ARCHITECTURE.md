# Infinite Memory Architecture

This document captures the intended "infinite memory" direction for Everlore:
event windows, client hygiene, Supermemory-style memory processing, calendars,
side-character chats, travel, locations, and long-running worlds.

## The Goal

A player should be able to play for 10,000-20,000+ turns and still feel that the
world remembers what happened, who was there, what promises were made or broken,
how relationships changed, how a place changed because of the player, what
emotional scars or loyalties remain, when an event happened in the story
calendar, and whether the event belongs to the main timeline, a flashback, or an
altered time-travel branch.

The model should not receive the whole transcript. The player can inspect the
whole transcript through Chronicle, but the model receives a compact context
packet derived from the transcript.

```text
Full transcript = durable player-visible history.
Prompt context = bounded, structured, emotionally-aware projection.
```

## Current Everlore Memory Stack

| Layer | Current role |
|-------|--------------|
| `events` | Durable story ledger, Chronicle history, rewind/replay source |
| recent events | Last raw turns used for short-term conversational continuity |
| scene summaries | Compressed chapter-like recaps for older raw turns |
| memories | Atomic long-term facts stored in Mongo + Pinecone |
| character codex | Structured protagonist/NPC cards with permanent facts and current state |
| world state / flags | Structured game-state snapshot |
| lore RAG | Template/world lore retrieved semantically |

Recent fixes clarified the active windows:

```text
Prompt raw event window: 6 turns
Play load window: 20 turns
Active client event cap: 100 turns
Active client memory cap: 50 memories
Local SQLite event cache: 200 turns per instance
Chronicle default page size: 50 turns
```

The full server event log remains durable. Active client state and model context
are bounded.

## Supermemory-Style Mapping

The researched Supermemory pattern describes long-term memory using contextual
resolution, atomic extraction, relationship/version graphing, dual-layer time
grounding, hybrid retrieval, profile packaging, and decay/storage tiers.

Mapped to Everlore:

| Supermemory concept | Everlore today | Intended upgrade |
|---------------------|----------------|------------------|
| Semantic chunking | Event-level processing | Scene/block-level extraction for long sequences |
| Contextual resolution | Partial: player narration facts and codex context | Rewrite memories with explicit names, locations, relationships, and story-time anchors |
| Atomic memories | Memory processor extracts 0-3 facts | Rich atoms with subjects, objects, emotional impact, status, and source events |
| Graph `updates` | Codex `retire_state` and memory supersession | Version all memory atoms and relationship edges |
| Graph `extends` | Codex facts accumulate implicitly | Explicit edges for facts that refine existing canon |
| Graph `derives` | Mostly absent | Inferred traits, relationship patterns, and long-term emotional arcs |
| Dual time | `sequence` and `created_at` | Story calendar time, event time, timeline branch, subjective time |
| Hybrid retrieval | Pinecone vector search | Vector + BM25 + entity graph + timeline filters |
| Static/dynamic profile | Codex + memories + recent turns | Explicit context packet with stable canon, current state, emotional continuity, and recent events |
| Smart decay | Basic memory archival/dedup | Tiered retrieval weights by importance, access, recency, and relationship relevance |

## Target Architecture

```text
1. Event Ledger
2. Entity Graph
3. Memory Atoms
4. Timeline / Calendar Layer
5. Retrieval Context Packet
```

### 1. Event Ledger

The event ledger is the source of truth. Every meaningful change can be modeled
as an event:

```ts
WorldEvent {
  id: ObjectId
  instance_id: ObjectId
  sequence: number
  branch_id: string
  parent_event_id?: ObjectId
  type:
    | "narration"
    | "travel"
    | "side_chat"
    | "calendar_tick"
    | "memory_edit"
    | "relationship_change"
    | "location_change"
  actor_entity_ids: ObjectId[]
  location_entity_id?: ObjectId
  story_time?: TimeAnchor
  real_created_at: Date
  data: Record<string, unknown>
  revision: number
  status: "active" | "superseded" | "deleted"
}
```

The invariant: projections must be traceable back to source events.

### 2. Entity Graph

Entities are the reusable nouns of the story:

- player
- protagonist
- side characters
- locations
- factions
- items
- quests
- relationships
- calendar eras
- timeline branches

Edges describe how they relate:

```text
Player --betrayed--> Mira
Mira --forgave--> Player
Player --traveled_to--> Ashen City
Ritual --happened_at--> Moon Temple
Memory B --updates--> Memory A
Memory C --extends--> Memory B
Memory D --derives_from--> Memory B + Memory C
```

This graph lets side-character chats, travel, cities, factions, romance arcs,
quests, and time travel share the same memory substrate.

### 3. Memory Atoms

Current memory rows are text + type + importance + vector. The target memory atom
should carry enough structure to support emotional continuity and exact recall:

```ts
MemoryAtom {
  id: ObjectId
  instance_id: ObjectId
  text: string
  type:
    | "fact"
    | "emotion"
    | "relationship"
    | "promise"
    | "location"
    | "quest"
    | "secret"
  subject_entity_ids: ObjectId[]
  object_entity_ids: ObjectId[]
  source_event_ids: ObjectId[]
  importance: number
  emotional_valence?: string
  emotional_cause?: string
  emotional_effect?: string
  relationship_delta?: string
  unresolved_thread?: boolean
  current_status: "current" | "superseded" | "archived"
  updates_memory_ids: ObjectId[]
  extends_memory_ids: ObjectId[]
  derives_from_memory_ids: ObjectId[]
  story_time?: TimeAnchor
  sequence_range: { start: number; end: number }
  embedding_id?: string
}
```

Weak memory:

```text
Mira forgave the player.
```

Strong memory:

```text
Mira chose to forgive the player after the ash bridge betrayal, but her trust
remains fragile; physical reassurance and honesty now matter deeply to her.
```

The second memory gives the model emotional instructions that still matter
thousands of turns later.

### 4. Timeline / Calendar Layer

Sequence order is not enough for immersive worlds. Everlore needs separate time
dimensions:

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

This handles normal calendars, magical calendars, flashbacks, prophecy, time
loops, alternate branches, side-character scenes happening elsewhere, and travel
time between cities.

Important distinction:

```text
Sequence = when the player experienced it.
Story time = when it happened in-world.
Timeline ID = which reality/branch it belongs to.
Causal edges = what caused what.
```

### 5. Retrieval Context Packet

Before every LLM call, build a structured packet rather than dumping arrays:

```text
CURRENT SCENE
- location
- active characters
- story date/time
- timeline branch

RECENT RAW EVENTS
- last few turns

RELEVANT LONG MEMORY
- vector + keyword + graph + timeline retrieval

RELATIONSHIP STATE
- trust, affection, fear, rivalry, unresolved wounds

CHARACTER CODEX
- permanent facts
- current mutable state
- private thoughts / dispositions

LOCATION CONTEXT
- what happened here before
- how the place changed

TIME CONTEXT
- relevant past, future, flashback, or alternate-branch facts
```

This is the layer that makes the world feel like it remembers.

## Retrieval Strategy

| Path | Best for |
|------|----------|
| Vector search | Conceptual similarity and emotional resonance |
| BM25 / keyword | Names, places, exact promises, artifacts, dates |
| Entity graph | Relationships, who affected whom, location history |
| Timeline filters | "Before the coronation", "in the altered timeline", "during the third moon" |
| Recency windows | Immediate conversational flow |
| Summary hierarchy | Older scene/chapter/arc continuity |

The retrieval system should return a ranked packet, not raw search results.

## Static vs Dynamic Context

```text
Static canon:
- protagonist identity
- world premise
- permanent character facts
- durable lore

Dynamic state:
- current location
- current relationships
- active conflicts
- unresolved promises
- recent turns
- relevant emotional scars
```

This mirrors static and dynamic profiles, adapted for interactive fiction.

## Design Invariants

1. Never delete player-visible history for context management.
2. Every projection must keep source provenance.
3. Edits/replays/rewinds must invalidate affected projections.
4. Memory retrieval must be structured, not blind vector dumping.
5. Story time and sequence time are different.
6. Emotional consequence is first-class memory.
7. Locations, side characters, factions, and calendars use the same graph.
8. The event ledger is canonical; projections are rebuildable.

