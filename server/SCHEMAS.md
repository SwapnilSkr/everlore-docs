# Everlore Server — Schema Reference

Canonical request/response shapes derived from `everlore-server/src/models/`, `src/schemas/`, and worker pub/sub payloads.

**Source of truth:** TypeScript models in the repo. When code and docs disagree, trust the code.

---

## ID conventions

| Context | Field | Format |
|---------|-------|--------|
| Mongo documents (HTTP) | `_id`, `*_id` refs | 24-char hex **ObjectId** string in JSON |
| Auth / shaped API responses | `id` | Same ObjectId hex (via `idString()`) |
| WebSocket pub/sub | `instanceId`, `eventId`, `userId` | camelCase string IDs |

Example IDs used below:

```
userId      = "674a1b2c3d4e5f6071829301"
templateId  = "674a1b2c3d4e5f6071829302"
instanceId  = "674a1b2c3d4e5f6071829303"
eventId     = "674a1b2c3d4e5f6071829304"
memoryId    = "674a1b2c3d4e5f6071829305"
characterId = "674a1b2c3d4e5f6071829306"
```

There are **no** `usr_`, `inst_`, `evt_`, or `mem_` prefixes.

---

## User

### `UserDoc` (Mongo)

```json
{
  "_id": "674a1b2c3d4e5f6071829301",
  "email": "user@example.com",
  "phone": null,
  "username": "player123",
  "password_hash": "…",
  "tier": "free",
  "providers": ["password"],
  "google_sub": null,
  "preferences": { … },
  "token_balance": 15000,
  "created_at": "2025-01-10T00:00:00.000Z",
  "updated_at": "2025-01-15T10:30:00.000Z"
}
```

`tier`: `"free"` | `"premium"` | `"creator"`

### `UserPreferences`

```json
{
  "nsfw_enabled": false,
  "preferred_model": "gpt-4o",
  "theme": "dark",
  "narration_length": "detailed",
  "auto_memory_curation": true,
  "player_name": "Aria",
  "gender": "female",
  "interests": ["fantasy", "romance"]
}
```

| Field | Notes |
|-------|-------|
| `player_name` | Optional; post-auth onboarding |
| `gender` | Optional: `"male"` \| `"female"` \| `"non_binary"` |
| `interests` | Optional genre keys from onboarding |
| `narration_length` | Validated on PUT: `"concise"` \| `"detailed"` \| `"verbose"` |

### Auth API shape (`serializeUser`)

Returned by `/auth/register`, `/auth/login`, `/auth/google`, `/auth/otp/verify`, `/auth/me`:

```json
{
  "id": "674a1b2c3d4e5f6071829301",
  "email": "user@example.com",
  "phone": null,
  "username": "player123",
  "tier": "free",
  "preferences": { … },
  "token_balance": 15000
}
```

Login/register also return `{ "token": "<jwt>" }`.

---

## World instance

### `WorldInstanceDoc`

```json
{
  "_id": "674a1b2c3d4e5f6071829303",
  "template_id": "674a1b2c3d4e5f6071829302",
  "template_version": 1,
  "player_id": "674a1b2c3d4e5f6071829301",
  "world_state": { "health": 85, "sanity": 92 },
  "active_flags": { "has_weapon": true },
  "current_scene": {
    "tag": "exploration",
    "turn_count": 7,
    "summary_pending": false
  },
  "narration_pov": "third",
  "mode": "free_play",
  "message_length": "medium",
  "focus_character_id": null,
  "current_time_anchor": { … },
  "active_timeline_id": "main",
  "default_calendar_id": "674a1b2c3d4e5f6071829307",
  "current_location": {
    "entity_id": "674a1b2c3d4e5f6071829308",
    "name": "The Rusty Anchor",
    "name_normalized": "the rusty anchor"
  },
  "persona_id": null,
  "persona_snapshot": null,
  "meta": {
    "total_events": 42,
    "total_memories": 15,
    "total_tokens_consumed": 150000,
    "last_active_at": "2025-01-15T10:30:00.000Z",
    "is_archived": false,
    "milestones": [{ "label": "First blood", "sequence": 12, "at": "…" }],
    "last_fate_seed_sequence": 38,
    "last_continuity_audit": null
  },
  "created_at": "2025-01-12T08:00:00.000Z",
  "updated_at": "2025-01-15T10:30:00.000Z"
}
```

### PATCH `/instances/:id/settings`

```json
{
  "narration_pov": "first",
  "mode": "free_play",
  "message_length": "short",
  "focus_character_id": "674a1b2c3d4e5f6071829306",
  "persona_id": null
}
```

| Field | Values |
|-------|--------|
| `narration_pov` | `"first"` \| `"third"` |
| `message_length` | `"short"` \| `"medium"` \| `"long"` |
| `focus_character_id` | ObjectId string or `null` |
| `persona_id` | ObjectId string or `null` |

---

## World event

### `WorldEventDoc`

```json
{
  "_id": "674a1b2c3d4e5f6071829304",
  "instance_id": "674a1b2c3d4e5f6071829303",
  "player_id": "674a1b2c3d4e5f6071829301",
  "sequence": 43,
  "type": "narration",
  "side_chat": null,
  "data": {
    "player_input": "*I examine the device*",
    "player_spoken_input": "",
    "player_narration_facts": ["I examine the device"],
    "ai_response": "The device pulses…",
    "choices": [
      { "label": "Touch it", "kind": "act", "send": "*I reach out to touch it*" },
      { "label": "Ask who built it", "kind": "say", "send": "Who built this?" }
    ],
    "milestone": null,
    "time_advanced": null,
    "travel": null,
    "fate_thread": null,
    "present_characters": ["Mira"],
    "codex_deltas": [],
    "replay_variants": [],
    "selected_replay_index": 0,
    "state_mutations": { "sanity": { "op": "subtract", "value": 2 } },
    "flag_mutations": { "has_weapon": { "op": "set", "value": true } },
    "model_used": "gpt-4o",
    "tokens_in": 1200,
    "tokens_out": 340
  },
  "is_user_edited": false,
  "edit_history": [],
  "scene_tag": "exploration",
  "time_anchor": { … },
  "location_anchor": { … },
  "created_at": "2025-01-15T10:30:00.000Z"
}
```

**Event types:** `"narration"` | `"intimate"` | `"calendar_tick"` | `"travel"` | `"side_chat"` | …

**Side chat events** add:

```json
{
  "type": "side_chat",
  "side_chat": {
    "character_id": "674a1b2c3d4e5f6071829306",
    "character_entity_id": "674a1b2c3d4e5f6071829309",
    "character_name": "Mira"
  }
}
```

---

## Memory

### `MemoryDoc` (Mongo; HTTP returns raw docs)

```json
{
  "_id": "674a1b2c3d4e5f6071829305",
  "instance_id": "674a1b2c3d4e5f6071829303",
  "player_id": "674a1b2c3d4e5f6071829301",
  "text": "The player showed kindness to the wounded guard",
  "type": "observation",
  "importance": 3,
  "is_nsfw": false,
  "source_event_ids": ["674a1b2c3d4e5f6071829304"],
  "pinecone_id": "674a1b2c3d4e5f6071829305",
  "access_count": 0,
  "last_accessed_at": "2025-01-15T10:30:00.000Z",
  "is_archived": false,
  "status": "active",
  "origin": "main",
  "known_by_entity_ids": [],
  "subjects": ["Player"],
  "objects": ["Guard"],
  "subject_entity_ids": [],
  "object_entity_ids": [],
  "time_anchor": { … },
  "timeline_id": "main",
  "location_anchor": null,
  "location_entity_id": null,
  "location_name": null,
  "emotional_valence": "tender",
  "emotional_cause": null,
  "emotional_effect": null,
  "relationship_delta": null,
  "unresolved_thread": false,
  "resolved_at": null,
  "created_at": "2025-01-15T10:30:00.000Z",
  "updated_at": "2025-01-15T10:30:00.000Z"
}
```

**Memory types** (edit validation): `"relationship"` | `"promise"` | `"lore"` | `"observation"` | `"emotion"` | `"secret"`

**Origin:** `"main"` (default) | `"side_chat"` — side-chat memories are scoped by `known_by_entity_ids`.

---

## Character codex

### `CharacterProfileDoc`

```json
{
  "_id": "674a1b2c3d4e5f6071829306",
  "instance_id": "674a1b2c3d4e5f6071829303",
  "player_id": "674a1b2c3d4e5f6071829301",
  "canonical_name": "Mira",
  "name_normalized": "mira",
  "aliases": [],
  "role": "Healer",
  "appearance": "Silver hair, green eyes",
  "persona": "Warm but guarded",
  "immutable_facts": ["Trained at the academy"],
  "mutable_state": ["Currently worried about the plague"],
  "disposition_to_player": "Cautiously friendly",
  "hidden_thought": "…",
  "relationship": {
    "trust": 55,
    "affection": 48,
    "fear": 0,
    "rivalry": 0
  },
  "entity_id": "674a1b2c3d4e5f6071829309",
  "is_protagonist": false,
  "first_seen_sequence": 3,
  "last_seen_sequence": 43,
  "mention_count": 12,
  "created_at": "…",
  "updated_at": "…"
}
```

Relationship meters are 0–100. Trust/affection start at 50; fear/rivalry start at 0.

---

## Time anchor

### `TimeAnchorDoc`

```json
{
  "real_time": "2025-01-15T10:30:00.000Z",
  "sequence": 43,
  "story_calendar": {
    "calendar_id": "674a1b2c3d4e5f6071829307",
    "year": 1247,
    "month": 3,
    "day": 14,
    "era": "Third Age",
    "label": "Spring equinox"
  },
  "event_time_label": "three days later",
  "timeline_id": "main",
  "causal_parent_event_ids": ["674a1b2c3d4e5f6071829303"],
  "subjective_entity_times": {}
}
```

---

## WebSocket — client → server

Connect: `WS /ws/play?token={jwt}`

Envelope (all actions):

```json
{
  "action": "chat",
  "instance_id": "674a1b2c3d4e5f6071829303",
  "event_id": "674a1b2c3d4e5f6071829304",
  "payload": { }
}
```

| Action | Required fields | Payload |
|--------|-----------------|---------|
| `chat` | `instance_id` | `{ "message": "…" }` — 1–4000 chars |
| `continue` | `instance_id` | `{ "advance"?: "hours" \| "day" \| "days" \| "season" }` — optional time skip |
| `side_chat` | `instance_id` | `{ "character_id": "…", "message": "…" }` |
| `replay` | `instance_id`, `event_id` | — |
| `load_instance` | `instance_id` | — |
| `ping` | — | — |

Immediate ack on dispatch:

```json
{ "type": "ack", "jobId": "42" }
```

Errors:

```json
{ "type": "error", "message": "Invalid message" }
{ "type": "error", "code": "RATE_LIMITED", "retryAfter": 45 }
{ "type": "error", "code": "GENERATION_IN_PROGRESS" }
```

---

## WebSocket — server → client

Relayed verbatim from Redis channel `user:{userId}:events`.

### Connection lifecycle

```json
{ "type": "connected", "userId": "674a1b2c3d4e5f6071829301" }
{ "type": "pong" }
{ "type": "account_deleted" }
```

### Main story generation

```json
{ "type": "generation_delta", "instanceId": "…", "delta": "The " }
{ "type": "generation_stream_end", "instanceId": "…", "narrative": "…full prose…" }
```

```json
{
  "type": "generation_complete",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "event": {
    "id": "674a1b2c3d4e5f6071829304",
    "sequence": 43,
    "narrative": "…",
    "scene_tag": "exploration",
    "emotional_tone": "curious",
    "model_used": "gpt-4o",
    "choices": [{ "label": "…", "kind": "act", "send": "…" }],
    "milestone": null,
    "present_characters": ["Mira"],
    "time_advanced": null,
    "time_anchor": { … },
    "location_anchor": { "entity_id": "…", "name": "…", "name_normalized": "…" },
    "fate_thread": null,
    "event_type": "narration",
    "state_diff": {
      "world_state": { "health": 85 },
      "active_flags": { "has_weapon": true }
    }
  }
}
```

```json
{
  "type": "generation_failed",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "message": "The world could not respond. Please try again."
}
```

```json
{
  "type": "milestone_unlocked",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "milestone": { "label": "First blood", "sequence": 43 }
}
```

### Side chat

```json
{ "type": "side_chat_delta", "instanceId": "…", "characterId": "…", "delta": "…" }
```

```json
{
  "type": "side_chat_complete",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "character": { "id": "674a1b2c3d4e5f6071829306", "name": "Mira" },
  "event": {
    "id": "674a1b2c3d4e5f6071829304",
    "sequence": 44,
    "player_input": "What do you know about the device?",
    "narrative": "…",
    "model_used": "gpt-4o",
    "created_at": "2025-01-15T10:31:00.000Z"
  }
}
```

```json
{
  "type": "side_chat_error",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "characterId": "674a1b2c3d4e5f6071829306",
  "message": "…"
}
```

### Replay

```json
{ "type": "replay_delta", "instanceId": "…", "eventId": "…", "delta": "…" }
```

```json
{
  "type": "replay_complete",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "eventId": "674a1b2c3d4e5f6071829304",
  "narrative": "…",
  "selected_index": 1,
  "choices": [{ "label": "…", "kind": "act", "send": "…" }],
  "present_characters": ["Mira"],
  "variants": [
    {
      "id": "v1",
      "narrative": "…",
      "model_used": "gpt-4o",
      "created_at": "…",
      "choices": [{ "label": "…", "kind": "say", "send": "…" }],
      "present_characters": ["Mira", "Kael"]
    }
  ]
}
```

Per-variant chips and presence are regenerated on replay and restored on variant select (no re-extraction).

### Post-turn projections

```json
{
  "type": "memories_curated",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "memories": [
    { "id": "674a1b2c3d4e5f6071829305", "text": "…", "type": "observation", "importance": 3 }
  ]
}
```

```json
{
  "type": "character_codex_updated",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "focused_character_id": null,
  "characters": [
    {
      "id": "674a1b2c3d4e5f6071829306",
      "canonical_name": "Mira",
      "aliases": [],
      "role": "Healer",
      "appearance": "…",
      "persona": "…",
      "immutable_facts": [],
      "mutable_state": [],
      "disposition_to_player": "…",
      "hidden_thought": "…",
      "relationship": { "trust": 55, "affection": 48, "fear": 0, "rivalry": 0 },
      "mention_count": 12,
      "is_protagonist": false
    }
  ]
}
```

### `load_instance` response

```json
{
  "type": "instance_loaded",
  "data": {
    "instance": { …WorldInstanceDoc… },
    "template": { …WorldTemplateDoc or null… },
    "recentEvents": [ …WorldEventDoc[] — main story only, no side_chat… ],
    "memories": [ …MemoryDoc[] top 20 by importance… ],
    "characters": [ …CharacterProfileDoc[] top 30… ],
    "eventWindow": {
      "limit": 30,
      "total": 42,
      "hasOlder": true
    }
  }
}
```

---

## Chronicle — shaped responses

These endpoints return **service-shaped** JSON (not always raw Mongo docs).

### GET `/chronicle/events/:instanceId`

```json
{
  "events": [ …WorldEventDoc[] oldest-first within page… ],
  "total": 42,
  "page": 1
}
```

Excludes `type: "side_chat"`.

### GET `/chronicle/recap/:instanceId`

```json
{
  "spine": "Scene summary prose or trimmed last turn…",
  "where": "The Rusty Anchor",
  "when": "three days later",
  "open_threads": [
    { "id": "674a1b2c3d4e5f6071829305", "text": "Find the lost amulet", "importance": 4 }
  ],
  "bonds": [
    {
      "id": "674a1b2c3d4e5f6071829306",
      "name": "Mira",
      "disposition": "Cautiously friendly",
      "meters": { "trust": 55, "affection": 48, "fear": 0, "rivalry": 0 }
    }
  ]
}
```

### GET `/chronicle/threads/:instanceId`

```json
{
  "open": [
    {
      "id": "674a1b2c3d4e5f6071829305",
      "text": "…",
      "type": "promise",
      "importance": 4,
      "emotional_valence": "anxious",
      "resolved_at": null,
      "time_anchor": { … }
    }
  ],
  "resolved": [ …same shape, resolved_at set… ]
}
```

### GET `/chronicle/relationships/:instanceId`

```json
{
  "characters": [
    {
      "id": "674a1b2c3d4e5f6071829306",
      "name": "Mira",
      "role": "Healer",
      "disposition": "Cautiously friendly",
      "meters": { "trust": 55, "affection": 48, "fear": 0, "rivalry": 0 },
      "mention_count": 12,
      "moments": [{ "label": "She confided in you", "sequence": 38 }]
    }
  ]
}
```

### GET `/chronicle/locations/:instanceId`

```json
{
  "current_location": { "entity_id": "674a1b2c3d4e5f6071829308", "name": "The Rusty Anchor" },
  "places": [
    {
      "entity_id": "674a1b2c3d4e5f6071829308",
      "name": "The Rusty Anchor",
      "event_count": 8,
      "memory_count": 3,
      "first_seen_sequence": 5,
      "last_seen_sequence": 43
    }
  ]
}
```

### GET `/chronicle/side-chats/:instanceId`

```json
{
  "threads": [
    {
      "character_id": "674a1b2c3d4e5f6071829306",
      "character_name": "Mira",
      "last_message": "…",
      "last_at": "2025-01-15T10:31:00.000Z",
      "turn_count": 4
    }
  ]
}
```

### POST `/chronicle/rewind/:instanceId`

Body: `{ "sequence": 30 }`

---

## Edit bodies

### PUT `/chronicle/memory/:memoryId`

```json
{
  "text": "Updated memory text",
  "type": "observation",
  "importance": 4
}
```

### PUT `/chronicle/event/:eventId`

```json
{
  "ai_response": "Updated response text",
  "player_input": "Updated input text"
}
```

**Response** (when narrative changes):

```json
{
  "success": true,
  "memories_deleted": 2,
  "recuration_queued": true,
  "choices": [{ "label": "…", "kind": "act", "send": "…" }],
  "present_characters": ["Mira"]
}
```

When only `player_input` is edited, `choices` and `present_characters` are `null` (chips unchanged).

### POST `/chronicle/replay/select/:eventId`

```json
{ "variant_index": 1 }
```
