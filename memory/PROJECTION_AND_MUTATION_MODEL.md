# Projection And Mutation Model

This document explains how the infinite-memory architecture should handle
player-driven mutation: replaying, selecting variants, deleting, editing events,
editing memories, rewinding, and resetting.

## Core Rule

```text
Events are source-of-truth.
All memory layers are projections.
When source events change, projections must be invalidated and rebuilt.
```

Projection examples:

- memory atoms
- Pinecone vectors
- character codex deltas
- relationship edges
- location state
- timeline/calendar entries
- scene/chapter/arc summaries
- retrieval indexes

## Mutation Types

### Event Edit

When a player edits an event response or player input:

```text
1. Save the edited event revision.
2. Mark projections sourced only from the old revision as stale.
3. Re-extract memories from the edited event.
4. Rebuild affected codex facts/state.
5. Rebuild affected relationship/location/timeline edges.
6. Re-embed changed memory atoms.
7. Mark summaries covering that sequence range as stale or requeue them.
```

Current Everlore already re-curates memories for edited events. The target
architecture extends that rule to every projection type.

### Replay / Select Replay Variant

Replay generates alternate prose for an existing event. Selection makes one
variant canonical.

Target behavior:

```text
1. Keep replay variants as event revision history.
2. Treat selected variant as the active event revision.
3. Stale projections from the old active revision.
4. Re-project from the selected revision.
5. If state or character facts changed, rebuild downstream ranges as needed.
```

For early implementation, rebuild projections sourced from that event. Later,
add range rebuilds for events after the changed turn if the selected variant
materially changes state.

### Rewind

Rewind removes a turn and everything after it from active canon.

Target behavior:

```text
1. Mark events >= sequence as inactive or delete them, depending on product policy.
2. Remove vectors/projections sourced from inactive events.
3. Delete or stale summaries covering removed ranges.
4. Recompute state, flags, codex, relationships, locations, and calendar state
   from surviving active events.
5. Bust session caches.
```

Everlore already does this for events, memories, scene summaries, character
codex, world state, and flags. The future graph should follow the same pattern.

### Event Delete

There are two kinds of delete:

| Delete type | Meaning |
|-------------|---------|
| Hard delete | Remove from storage, usually for user deletion/privacy |
| Story delete / tombstone | Remove from active canon while retaining audit/revision history |

For normal story correction, prefer tombstone/supersede because it makes rebuild
logic easier and protects provenance. For privacy/account deletion, hard delete.

Target behavior:

```text
1. Remove or deactivate the event.
2. Remove source_event_id from projections with multiple sources.
3. Delete/stale projections that have no remaining active sources.
4. Rebuild affected summaries and graph edges.
```

### Memory Edit

When the player edits a memory:

```text
1. Update memory text.
2. Re-embed the same memory or create a new revision.
3. Re-resolve linked entities if the text changed materially.
4. If removed facts contradict current graph state, trigger supersession.
5. Preserve source provenance if still valid.
```

Everlore already re-embeds edited memories against the same vector id. The target
architecture adds entity relinking and graph supersession.

### Character Edit

When a player edits a character card:

```text
1. Update structured fields.
2. Detect removed immutable/current facts.
3. Supersede memories and edges that only support removed facts.
4. Keep the edited card high-authority for future prompts.
```

Everlore already supersedes memory vectors when character edits remove facts.

## Projection Records

Every projection should store:

```ts
ProjectionMetadata {
  source_event_ids: ObjectId[]
  source_event_revisions?: number[]
  generated_by: "memory_processor" | "codex_extractor" | "summary_processor" | string
  generated_at: Date
  status: "active" | "stale" | "superseded" | "archived"
}
```

This enables reliable rebuilds.

## Range Rebuilds

Some edits affect only one event. Others affect downstream canon.

Examples:

- Editing a typo in prose: rebuild only sourced memories.
- Changing "Mira accepts" to "Mira refuses": rebuild relationship, state, and
  later summaries.
- Replaying a battle outcome: rebuild world state and downstream codex facts.
- Rewinding: rebuild from the surviving event range.

Start conservative:

```text
single-event rebuild first
range rebuild for rewind/reset
manual/admin repair tools for larger inconsistencies
```

Then evolve toward automatic range rebuilds for high-impact mutations.

## Repair Jobs

The system should have background repair jobs for:

- stale summaries
- missing vectors
- memory atoms without entity links
- relationship edges pointing to inactive events
- codex facts sourced only from inactive/deleted turns
- calendar entries with missing source events

Repair jobs should be idempotent and safe to rerun.

