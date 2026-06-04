# Everlore Infinite Memory

This folder documents the intended long-horizon memory architecture for
Everlore: how a player can chat for 10,000-20,000+ turns while the world still
feels like it remembers what happened, who was changed by it, where it happened,
and when it happened in story time.

The goal is not to send every message back to the model. The goal is to keep a
durable story ledger and continuously project it into compact, accurate,
emotionally-aware context.

## Documents

| Document | Purpose |
|----------|---------|
| [MEMORY_ARCHITECTURE.md](MEMORY_ARCHITECTURE.md) | Core infinite-memory architecture: current system, target system, and Supermemory-style mapping |
| [PROJECTION_AND_MUTATION_MODEL.md](PROJECTION_AND_MUTATION_MODEL.md) | How edits, replays, rewinds, deletes, and memory edits affect derived memory state |
| [CALENDAR_TIMELINES_AND_LOCATIONS.md](CALENDAR_TIMELINES_AND_LOCATIONS.md) | Story calendars, magical calendars, time travel, locations, and travel-aware memory |
| [FEATURES_ON_TOP.md](FEATURES_ON_TOP.md) | Product features enabled by the memory graph |
| [CHECKLIST.md](CHECKLIST.md) | Implementation checklist and phased roadmap |

## Core Principle

```text
Events are the source of truth.
Memories, summaries, codex cards, relationships, locations, calendars, and
retrieval packets are projections.
```

This lets Everlore preserve exact player-visible history while giving the model
a small, high-signal context packet each turn.

