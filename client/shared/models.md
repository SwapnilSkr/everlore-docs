# Shared Models

Data models used across multiple features. All models are plain Dart classes with `fromJson` factory constructors for API deserialization.

---

## User

**File:** `lib/shared/models/user.dart`

Represents an authenticated user.

### User

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `String` | required | Unique user ID |
| `email` | `String` | required | User email address |
| `phone` | `String?` | `null` | E.164 phone number for OTP-authenticated accounts |
| `username` | `String` | required | Display name |
| `tier` | `String` | required | Subscription tier (e.g., "free") |
| `preferences` | `UserPreferences` | `const UserPreferences()` | User settings |
| `tokenBalance` | `int?` | `null` | Remaining AI token balance |

**JSON Keys:** `id`, `email`, `phone`, `username`, `tier`, `preferences`, `token_balance`

### UserPreferences

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `nsfwEnabled` | `bool` | `false` | Whether NSFW content is enabled |
| `preferredModel` | `String` | `'gpt-4o'` | AI model selection |
| `theme` | `String` | `'dark'` | UI theme preference |
| `narrationLength` | `String` | `'detailed'` | AI response verbosity |
| `autoMemoryCuration` | `bool` | `true` | Automatic memory management |

**JSON Keys:** `nsfw_enabled`, `preferred_model`, `theme`, `narration_length`, `auto_memory_curation`

Both classes have `toJson()` methods for serialization.

---

## GameEvent

**File:** `lib/shared/models/event.dart`

Represents a single turn in the game — a player input and/or AI narrative response.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `String` | required | Event UUID |
| `instanceId` | `String` | required | Parent world instance |
| `sequence` | `int` | required | Ordering number |
| `type` | `String` | required | Event type (e.g., `narration`, `player_action`) |
| `playerInput` | `String?` | `null` | Player's message text |
| `aiResponse` | `String?` | `null` | AI narrative response |
| `sceneTag` | `String?` | `null` | Scene classification (combat, dialogue, etc.) |
| `stateMutations` | `Map<String, dynamic>?` | `null` | World state changes from this event |
| `flagMutations` | `Map<String, dynamic>?` | `null` | Flag changes from this event |
| `emotionalTone` | `String?` | `null` | AI-detected emotional tone |
| `createdAt` | `DateTime` | required | When the event occurred |
| `isOptimistic` | `bool` | `false` | Whether this is a provisional local-only event |
| `isUserEdited` | `bool` | `false` | Whether the player has edited this event |

### Factory Constructors

| Constructor | Purpose |
|-------------|---------|
| `fromJson(json)` | Deserializes from API response. Handles nested `data` object and multiple key formats (`id`/`_id`, `ai_response`/`narrative`) |
| `optimistic(instanceId, playerInput)` | Creates a provisional event with id `optimistic_<timestamp>` and sequence `-1` |
| `fromSqlite(row)` | Deserializes from SQLite row map |

### Serialization
- `toSqlite()` — converts to SQLite-compatible map for local database storage

### JSON Key Compatibility
`fromJson` handles multiple API response formats:
```dart
// Direct format
{ "id": "...", "ai_response": "...", "player_input": "..." }

// Nested data format
{ "id": "...", "data": { "ai_response": "...", "player_input": "..." } }

// Alternative keys
{ "_id": "...", "narrative": "...", "instanceId": "..." }
```

---

## Memory

**File:** `lib/shared/models/memory.dart`

Represents a curated memory — a piece of information the AI remembers about the game world.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `String` | required | Memory UUID |
| `instanceId` | `String` | required | Parent world instance |
| `text` | `String` | required | Memory content |
| `type` | `String` | required | Classification: `relationship`, `promise`, `lore`, `observation`, `emotion`, `secret` |
| `importance` | `int` | `3` | Importance rating (1-5) |
| `isNsfw` | `bool` | `false` | NSFW content flag |
| `isArchived` | `bool` | `false` | Whether memory is archived |
| `sourceEventIds` | `List<String>` | `[]` | Events that contributed to this memory |
| `accessCount` | `int` | `0` | How many times this memory was referenced |
| `createdAt` | `DateTime?` | `null` | When the memory was created |

**JSON Keys:** `_id`/`id`, `instance_id`, `text`, `type`, `importance`, `is_nsfw`, `is_archived`, `source_event_ids`, `access_count`, `created_at`

### Methods
- `copyWith({text, type, importance})` — creates modified copy (used in memory editing)

---

## WorldInstance

**File:** `lib/shared/models/world_instance.dart`

Represents a live game world instance — a player's running session of a template.

### WorldInstance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `String` | required | Instance UUID |
| `templateId` | `String` | required | Source template ID |
| `templateVersion` | `int` | `1` | Template version at creation time |
| `playerId` | `String` | required | Owning player's ID |
| `worldState` | `Map<String, num>` | `{}` | Current world stat values (health, mana, etc.) |
| `activeFlags` | `Map<String, dynamic>` | `{}` | Boolean/string flags (quest states, etc.) |
| `currentScene` | `SceneInfo` | `const SceneInfo()` | Current scene context |
| `meta` | `InstanceMeta` | `const InstanceMeta()` | Statistics and metadata |
| `createdAt` | `DateTime?` | `null` | Creation timestamp |
| `template` | `Map<String, dynamic>?` | `null` | Enriched template data from list endpoint |

### Methods
- `applyStateDiff(diff)` — merges state mutations and flag updates into a new `WorldInstance`

**JSON Keys:** `_id`/`id`, `template_id`, `template_version`, `player_id`, `world_state`, `active_flags`, `current_scene`, `meta`, `created_at`, `template`

### SceneInfo

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tag` | `String` | `'dialogue'` | Current scene type |
| `turnCount` | `int` | `0` | Turns in current scene |
| `summaryPending` | `bool` | `false` | Whether scene needs summarization |

### InstanceMeta

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `totalEvents` | `int` | `0` | Total event count |
| `totalMemories` | `int` | `0` | Total memory count |
| `totalTokensConsumed` | `int` | `0` | AI tokens used |
| `lastActiveAt` | `DateTime?` | `null` | Last activity timestamp |
| `isArchived` | `bool` | `false` | Archive status |

---

## WorldTemplate

**File:** `lib/shared/models/world_template.dart`

Represents a published world template that players can instantiate.

### WorldTemplate

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `String` | required | Template UUID |
| `creatorId` | `String` | required | Creator's user ID |
| `title` | `String` | required | Display title |
| `slug` | `String` | required | URL-safe slug |
| `description` | `String` | required | Template description |
| `isPublished` | `bool` | `false` | Whether template is publicly visible |
| `isSentient` | `bool` | `false` | Sentient NPC world vs game-master world |
| `isNsfwCapable` | `bool` | `false` | NSFW content support |
| `version` | `int` | `1` | Template version |
| `seedPrompt` | `String` | `''` | Initial prompt for new instances |
| `globalLore` | `String` | `''` | Background lore text |
| `baseStatsTemplate` | `Map<String, StatDefinition>` | `{}` | Stat definitions |
| `flagDefinitions` | `Map<String, dynamic>` | `{}` | Boolean flag definitions |
| `sceneTags` | `List<String>` | `[]` | Available scene types |
| `modelPreferences` | `Map<String, String>` | `{}` | AI model configuration |
| `maxContextMemories` | `int` | `25` | Max memories in AI context |
| `maxLoreResults` | `int` | `10` | Max lore retrieval results |
| `createdAt` | `DateTime?` | `null` | Creation timestamp |

### StatDefinition

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `defaultValue` | `num` | required | Starting value for new instances |
| `min` | `num` | required | Minimum value |
| `max` | `num` | required | Maximum value |
| `description` | `String` | required | Human-readable stat description |

`StatDefinition` has both `fromJson()` and `toJson()` methods.
