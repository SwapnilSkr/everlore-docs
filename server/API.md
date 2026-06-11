# Everlore Server — API Documentation

> **Schema reference:** [SCHEMAS.md](./SCHEMAS.md) — canonical field names and JSON shapes from the TypeScript models.  
> IDs are 24-char MongoDB **ObjectId** hex strings (`674a1b2c3d4e5f6071829301`), not prefixed strings.  
> For product behavior, see [../system-guide/](../system-guide/README.md). For every route file, see [../code-reference/SERVER.md](../code-reference/SERVER.md).

## Base URL

```
http://localhost:3000
```

## Authentication

Most endpoints require a Bearer token in the Authorization header:

```
Authorization: Bearer {jwt_token}
```

Tokens are obtained from `/auth/login`, `/auth/register`, `/auth/google`, or `/auth/otp/verify`.

## Content Types

All requests and responses use JSON:
```
Content-Type: application/json
```

## Request handling

HTTP and WebSocket paths are registered in `src/routes/`; each handler delegates to a matching controller in `src/controllers/`, which calls `src/services/` for persistence and side effects. See [SERVICES.md](./SERVICES.md) for the full map.

---

## HTTP Endpoints

### Health Check

#### GET /

Returns API status.

**Response:**
```json
"Everlore API"
```

#### GET /health

Returns detailed health status.

**Response:**
```json
{
  "ok": true,
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

---

### Authentication Routes (`/auth`)

#### POST /auth/register

Register a new user account.

**Body:**
```json
{
  "email": "user@example.com",
  "username": "player123",
  "password": "securePassword123"
}
```

**Validation:**
- `email`: Valid email format
- `username`: 3-30 characters, alphanumeric + underscore only
- `password`: 8-128 characters

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "674a1b2c3d4e5f6071829301",
    "email": "user@example.com",
    "phone": null,
    "username": "player123",
    "tier": "free",
    "preferences": {
      "nsfw_enabled": false,
      "preferred_model": "gpt-4o",
      "theme": "dark",
      "narration_length": "detailed",
      "auto_memory_curation": true
    },
    "token_balance": 15000
  }
}
```

**Errors:**
- `400` - Invalid input format
- `409` - Email or username already exists

---

#### POST /auth/login

Authenticate existing user.

**Body:**
```json
{
  "email": "user@example.com",
  "password": "securePassword123"
}
```

**Response:** Same as `/auth/register`

**Rate Limiting:** 10 attempts per 5 minutes per email

**Errors:**
- `401` - Invalid credentials
- `429` - Too many attempts (retry after shown in headers)

---

#### POST /auth/google

Authenticate via Google ID token.

**Body:**
```json
{
  "id_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

The backend verifies the Google token with Google's tokeninfo endpoint and, when `GOOGLE_CLIENT_ID` is configured, checks the token audience against it.

**Response:** Same as `/auth/register`

**User Creation:**
If user doesn't exist, auto-creates with:
- Email from token
- Username: `{email_prefix}_{timestamp}`
- Password: empty (OAuth-only)
- `google_sub` set from the verified Google subject

---

#### POST /auth/otp/send

Send an SMS verification code using Twilio Verify.

**Body:**
```json
{
  "phone": "+15551234567"
}
```

**Validation:**
- `phone`: E.164 format, `+` followed by 8-15 digits

**Response:**
```json
{
  "success": true,
  "mockCode": "123456"
}
```

**Notes:**
- In Twilio mock mode (`TWILIO_ACCOUNT_SID=AC_MOCK_SID`), no SMS is sent and `123456` is accepted.
- `mockCode` is returned by the current implementation for local development convenience.

**Rate Limiting:** 5 requests per 10 minutes per phone number

---

#### POST /auth/otp/verify

Verify an SMS code and exchange it for the Everlore JWT.

**Body:**
```json
{
  "phone": "+15551234567",
  "code": "123456"
}
```

**Response:** Same shape as `/auth/register` (includes `phone`, `token_balance`).

**Rate Limiting:** 10 requests per 10 minutes per phone number

**User Creation:**
- If the phone number is new, a user is auto-created with `providers: ['phone']`
- Existing users are updated to include the `phone` provider

---

#### GET /auth/me

Get current user profile.

**Headers:** `Authorization: Bearer {token}`

**Response:** `serializeUser` shape — see [SCHEMAS.md](./SCHEMAS.md#auth-api-shape-serializeuser).

---

#### PUT /auth/preferences

Update user preferences.

**Headers:** `Authorization: Bearer {token}`

**Body:** (all fields optional)
```json
{
  "nsfw_enabled": true,
  "preferred_model": "gpt-4o-mini",
  "theme": "light",
  "narration_length": "verbose",
  "auto_memory_curation": false,
  "player_name": "Aria",
  "gender": "female",
  "interests": ["fantasy", "romance"]
}
```

**Validation:**
- `narration_length`: `"concise"` | `"detailed"` | `"verbose"`
- `gender`: `"male"` | `"female"` | `"non_binary"`
- `player_name`: 2–40 characters

**Response:**
```json
{ "success": true }
```

---

### Template Routes (`/templates`)

#### GET /templates

List published templates (public).

**Query Parameters:**
- `page` (number, optional): Page number, default 1
- `limit` (number, optional): Items per page, default 20
- `search` (string, optional): Search in title and description

**Response:**
```json
{
  "templates": [
    {
      "_id": "674a1b2c3d4e5f6071829302",
      "creator_id": "674a1b2c3d4e5f6071829301",
      "title": "Mystic Academy",
      "slug": "mystic-academy",
      "description": "A magical school adventure...",
      "is_published": true,
      "is_sentient": false,
      "is_nsfw_capable": true,
      "version": 1,
      "seed_prompt": "You are a student at...",
      "global_lore": "The academy was founded...",
      "base_stats_template": { ... },
      "flag_definitions": { ... },
      "scene_tags": ["dialogue", "combat", "intimate"],
      "model_preferences": { ... },
      "max_context_memories": 25,
      "max_lore_results": 10,
      "created_at": "2025-01-10T00:00:00Z",
      "updated_at": "2025-01-10T00:00:00Z"
    }
  ],
  "total": 150,
  "page": 1
}
```

---

#### GET /templates/:id

Get single template by ID (public).

**Response:** Single template object (same structure as list items)

**Errors:**
- `404` - Template not found

---

#### POST /templates

Create new template (authenticated, creator/premium only).

**Headers:** `Authorization: Bearer {token}`

**Rate Limiting:** 5 templates per 24 hours

**Body:**
```json
{
  "title": "Space Station Omega",
  "description": "A sci-fi survival horror...",
  "is_sentient": false,
  "is_nsfw_capable": true,
  "seed_prompt": "You wake up in a cryo-pod...",
  "global_lore": "Station Omega was abandoned in 2187...",
  "base_stats_template": {
    "health": {
      "default": 100,
      "min": 0,
      "max": 100,
      "description": "Physical condition"
    },
    "sanity": {
      "default": 100,
      "min": 0,
      "max": 100,
      "description": "Mental stability"
    }
  },
  "flag_definitions": {
    "has_weapon": {
      "type": "boolean",
      "default": false,
      "description": "Player has a weapon"
    },
    "doors_unlocked": {
      "type": "integer",
      "default": 0,
      "description": "Number of doors unlocked"
    }
  },
  "scene_tags": ["exploration", "combat", "existential"],
  "model_preferences": {
    "logic": "gpt-4o",
    "narration_nsfw": "pygmalionai/mythalion-13b",
    "narration_sfw": "gpt-4o",
    "summary": "gpt-4o-mini"
  },
  "max_context_memories": 25,
  "max_lore_results": 10
}
```

**Validation:**
- `title`: 1-200 characters
- `description`: 1-2000 characters
- `seed_prompt`: 10-10000 characters
- `global_lore`: Max 50000 characters
- `max_context_memories`: 5-50
- `max_lore_results`: 3-20

**Response:** Created template object

**Errors:**
- `401` - Unauthorized
- `403` - Creator/premium tier required
- `429` - Rate limit exceeded

---

#### PUT /templates/:id

Update template (owner only).

**Body:** Partial template object (same fields as create, all optional)

**Response:** Updated template object

**Errors:**
- `404` - Template not found or not owned by user

---

#### POST /templates/:id/publish

Publish template (owner only).

**Effects:**
- Sets `is_published: true`
- Increments version
- Embeds and upserts `global_lore` to Pinecone

**Response:**
```json
{ "success": true }
```

---

#### GET /templates/mine/list

List current user's templates (creator only).

**Response:** Array of template objects (includes unpublished)

---

### Instance Routes (`/instances`)

#### GET /instances

List current user's instances.

**Query Parameters:**
- `page` (number, optional)
- `limit` (number, optional)
- `include_archived` (boolean, optional): Include soft-deleted instances

**Response:**
```json
{
  "instances": [
    {
      "_id": "674a1b2c3d4e5f6071829303",
      "template_id": "674a1b2c3d4e5f6071829302",
      "template_version": 1,
      "player_id": "674a1b2c3d4e5f6071829301",
      "world_state": {
        "health": 85,
        "sanity": 92
      },
      "active_flags": {
        "has_weapon": true,
        "doors_unlocked": 3
      },
      "current_scene": {
        "tag": "exploration",
        "turn_count": 7,
        "summary_pending": false
      },
      "narration_pov": "third",
      "mode": "free_play",
      "message_length": "medium",
      "focus_character_id": null,
      "current_time_anchor": null,
      "current_location": null,
      "meta": {
        "total_events": 42,
        "total_memories": 15,
        "total_tokens_consumed": 150000,
        "last_active_at": "2025-01-15T10:30:00Z",
        "is_archived": false,
        "milestones": []
      },
      "template": {
        "_id": "674a1b2c3d4e5f6071829302",
        "title": "Space Station Omega",
        "is_sentient": false,
        "description": "A sci-fi survival horror..."
      },
      "created_at": "2025-01-12T08:00:00Z",
      "updated_at": "2025-01-15T10:30:00Z"
    }
  ]
}
```

Full field list: [SCHEMAS.md — World instance](./SCHEMAS.md#world-instance).

#### PATCH /instances/:id/settings

Update per-instance play settings.

**Body:** (all optional)
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
| `focus_character_id` | ObjectId or `null` |

---

#### GET /instances/:id

Get single instance by ID.

**Response:** Instance object (same structure as list)

---

#### POST /instances

Create new instance from template.

**Body:**
```json
{
  "template_id": "674a1b2c3d4e5f6071829302"
}
```

**Validation:**
- Template must exist and be published
- User must be within tier instance limits

**Response:**
```json
{
  "instance": { ... },
  "template": { ... }
}
```

**Errors:**
- `404` - Template not found or not published
- `403` - Instance limit reached

---

#### POST /instances/:id/archive

Archive (soft-delete) an instance.

**Effects:**
- Sets `meta.is_archived: true`
- Clears Redis session cache

**Response:**
```json
{ "success": true }
```

---

### Chronicle Routes (`/chronicle`)

All routes require auth. Ownership verified via `world_instances` (`player_id` match).

#### Read surfaces (Lore Tome)

| Method | Path | Purpose | Key query params |
|--------|------|---------|------------------|
| GET | `/chronicle/events/:instanceId` | Timeline tab | `page`, `limit`, `type` — excludes `side_chat` |
| GET | `/chronicle/memories/:instanceId` | Echoes tab | `q` (full-text), `type`, `min_importance`, `unresolved`, `include_archived` |
| GET | `/chronicle/recap/:instanceId` | Recap landing | — |
| GET | `/chronicle/threads/:instanceId` | Threads tab | open + recently resolved |
| GET | `/chronicle/relationships/:instanceId` | Bonds tab | codex meters + narrative edges |
| GET | `/chronicle/relationships/:instanceId/:characterId/memories` | Character memory view | entity-linked memories |
| GET | `/chronicle/locations/:instanceId` | Places index | current location pinned first |
| GET | `/chronicle/locations/:instanceId/:locationEntityId` | Place journal | facts, state, events, memories |
| GET | `/chronicle/calendar/:instanceId` | Almanac | calendars, timelines, dated events |
| GET | `/chronicle/side-chats/:instanceId` | Side chat thread list | per-character aggregates |
| GET | `/chronicle/side-chats/:instanceId/:characterId` | One side-chat thread | paginated turns |

#### Calendar / timeline mutations

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/chronicle/calendar/:instanceId/timeline` | Fork a new timeline branch |
| PUT | `/chronicle/calendar/:instanceId/timeline/active` | Switch active reality |
| PUT | `/chronicle/calendar/event/:eventId/time-anchor` | Flashback: change story date without changing sequence |

#### Player edits & rollback

| Method | Path | Purpose |
|--------|------|---------|
| PUT | `/chronicle/memory/:memoryId` | Edit memory (re-embeds vector) |
| DELETE | `/chronicle/memory/:memoryId` | Delete memory + vector |
| PUT | `/chronicle/event/:eventId` | Edit turn → re-curates memories, stales summaries |
| PUT | `/chronicle/character/:characterId` | Edit codex card → may supersede memories |
| POST | `/chronicle/replay/:eventId` | Generate replay variants (also via WS) |
| POST | `/chronicle/replay/select/:eventId` | Commit a replay variant |
| POST | `/chronicle/rewind/:instanceId` | Roll back to sequence — body: `{ "sequence": N }` |

#### GET /chronicle/events/:instanceId

Paginated main-story event history (oldest first). Excludes `side_chat`.

**Query:** `page` (default 1), `limit` (default 50), optional `type` filter (ignored if `side_chat`).

**Response:**
```json
{
  "events": [ …WorldEventDoc[], oldest first within page… ],
  "total": 42,
  "page": 1
}
```

See [SCHEMAS.md — World event](./SCHEMAS.md#world-event).

---

#### GET /chronicle/recap/:instanceId

Deterministic "Story so far" card — no LLM call.

**Response:** See [SCHEMAS.md — recap](./SCHEMAS.md#get-chroniclerecapinstanceid).

---

#### GET /chronicle/threads/:instanceId

Open and recently resolved promise/conflict threads.

**Response:** `{ "open": [...], "resolved": [...] }` — shaped memory rows.

---

#### GET /chronicle/relationships/:instanceId

Bond ledger: codex meters + narrative edge moments.

**Response:** See [SCHEMAS.md — relationships](./SCHEMAS.md#get-chroniclerelationshipsinstanceid).

---

#### GET /chronicle/locations/:instanceId

Places index with current location pinned.

**Response:** See [SCHEMAS.md — locations](./SCHEMAS.md#get-chroniclerocationsinstanceid).

---

#### GET /chronicle/side-chats/:instanceId

One row per character with side-chat history.

**Response:** See [SCHEMAS.md — side chats](./SCHEMAS.md#get-chronicleside-chatsinstanceid).

---

#### GET /chronicle/memories/:instanceId

Main-story-visible memories. Supports Echoes search/filters.

**Query:** `q`, `type`, `min_importance`, `unresolved`, `include_archived`

---

#### PUT /chronicle/memory/:memoryId

Edit a memory.

**Body:**
```json
{
  "text": "Updated memory text",
  "type": "observation",
  "importance": 4
}
```

**Effects:**
- Updates MongoDB document
- Re-embeds and updates Pinecone vector

**Response:**
```json
{ "success": true }
```

---

#### DELETE /chronicle/memory/:memoryId

Delete a memory.

**Effects:**
- Deletes from MongoDB
- Deletes vector from Pinecone
- Decrements instance memory count

**Response:**
```json
{ "success": true }
```

---

#### PUT /chronicle/event/:eventId

Edit an event.

**Body:**
```json
{
  "ai_response": "Updated response text",
  "player_input": "Updated input text"
}
```

**Effects:**
- Saves previous data to `edit_history`
- Sets `is_user_edited: true`
- When `ai_response` changes: regenerates `choices` + `present_characters`, stores on new `edit` variant, re-curates memories

**Response:**
```json
{
  "success": true,
  "memories_deleted": 2,
  "recuration_queued": true,
  "choices": [{ "label": "…", "kind": "act", "send": "…" }],
  "present_characters": ["Mira"]
}
```

`choices` and `present_characters` are `null` when only `player_input` was edited.

---

## WebSocket Protocol

Full message catalog: [SCHEMAS.md — WebSocket](./SCHEMAS.md#websocket--client--server).

### Connection

```
WS /ws/play?token={jwt_token}
```

Token must be provided as query parameter (not header).

### Authentication Flow

1. Client connects with token
2. Server validates JWT
3. Server subscribes to `user:{userId}:events` Redis channel (first socket only)
4. Server sends: `{ "type": "connected", "userId": "674a1b2c3d4e5f6071829301" }`

### Client → Server Messages

All messages use this envelope:

```json
{
  "action": "chat",
  "instance_id": "674a1b2c3d4e5f6071829303",
  "event_id": "674a1b2c3d4e5f6071829304",
  "payload": { }
}
```

| Action | Required | Payload |
|--------|----------|---------|
| `chat` | `instance_id` | `{ "message": "…" }` — 1–4000 chars |
| `continue` | `instance_id` | `{ "advance"?: "hours" \| "day" \| "days" \| "season" }` — world advances without player input |
| `side_chat` | `instance_id` | `{ "character_id": "…", "message": "…" }` — private character thread |
| `replay` | `instance_id`, `event_id` | — streams alternative turn |
| `load_instance` | `instance_id` | — hydrates play screen |
| `ping` | — | keepalive |

On successful dispatch, server immediately sends:

```json
{ "type": "ack", "jobId": "42" }
```

**Error cases:**
```json
{ "type": "error", "message": "Invalid message" }
{ "type": "error", "code": "RATE_LIMITED", "retryAfter": 45 }
{ "type": "error", "code": "GENERATION_IN_PROGRESS" }
```

Rate limit: 10 chat-class actions per minute per user.

---

### Server → Client Events

Relayed verbatim from Redis pub/sub to all open sockets for the user.

#### Main story streaming

```json
{ "type": "generation_delta", "instanceId": "674a1b2c3d4e5f6071829303", "delta": "The " }
{ "type": "generation_stream_end", "instanceId": "674a1b2c3d4e5f6071829303", "narrative": "…full prose…" }
```

#### `generation_complete`

```json
{
  "type": "generation_complete",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "event": {
    "id": "674a1b2c3d4e5f6071829304",
    "sequence": 43,
    "narrative": "The device pulses with an eerie blue light…",
    "scene_tag": "exploration",
    "emotional_tone": "curious",
    "model_used": "gpt-4o",
    "choices": [
      { "label": "Touch it", "kind": "act", "send": "*I reach out to touch it*" }
    ],
    "milestone": null,
    "present_characters": ["Mira"],
    "time_advanced": null,
    "time_anchor": { "sequence": 43, "timeline_id": "main", "…": "…" },
    "location_anchor": { "entity_id": "674a1b2c3d4e5f6071829308", "name": "The Rusty Anchor", "name_normalized": "the rusty anchor" },
    "fate_thread": null,
    "event_type": "narration",
    "state_diff": {
      "world_state": { "health": 85, "sanity": 90 },
      "active_flags": { "has_weapon": true }
    }
  }
}
```

#### `generation_failed`

Permanent job failure (after retries):

```json
{
  "type": "generation_failed",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "message": "The world could not respond. Please try again."
}
```

#### Side chat

```json
{ "type": "side_chat_delta", "instanceId": "…", "characterId": "674a1b2c3d4e5f6071829306", "delta": "…" }
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

#### Replay

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
  "variants": [{
    "id": "v1",
    "narrative": "…",
    "model_used": "gpt-4o",
    "created_at": "…",
    "choices": [{ "label": "…", "kind": "say", "send": "…" }],
    "present_characters": ["Mira"]
  }]
}
```

#### Post-turn projections

```json
{
  "type": "memories_curated",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "memories": [
    { "id": "674a1b2c3d4e5f6071829305", "text": "The player showed kindness…", "type": "observation", "importance": 3 }
  ]
}
```

```json
{
  "type": "character_codex_updated",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "focused_character_id": null,
  "characters": [ …full codex cards… ]
}
```

```json
{
  "type": "milestone_unlocked",
  "instanceId": "674a1b2c3d4e5f6071829303",
  "milestone": { "label": "First blood", "sequence": 43 }
}
```

#### `instance_loaded`

```json
{
  "type": "instance_loaded",
  "data": {
    "instance": { …WorldInstanceDoc… },
    "template": { …WorldTemplateDoc or null… },
    "recentEvents": [ …main story only, no side_chat… ],
    "memories": [ …top 20 by importance… ],
    "characters": [ …top 30 codex cards… ],
    "eventWindow": { "limit": 30, "total": 42, "hasOlder": true }
  }
}
```

#### Other

```json
{ "type": "pong" }
{ "type": "account_deleted" }
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET/PUT/POST |
| 201 | Created | Successful resource creation |
| 400 | Bad Request | Invalid input data |
| 401 | Unauthorized | Missing or invalid token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server error |

### Error Response Format

```json
{
  "error": "Human-readable error message"
}
```

### Rate Limit Headers

When rate limited (429), response includes:
```
Retry-After: 45  // Seconds until retry allowed
```

---

## Rate Limits

| Action | Limit | Window |
|--------|-------|--------|
| Login attempts | 10 | 5 minutes |
| Chat messages | 10 | 1 minute |
| Template creation | 5 | 24 hours |
| Memory edits | 30 | 1 hour |

---

## CORS

The API supports CORS for configured origins:

```
Access-Control-Allow-Origin: {configured_origin}
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

Configure origins via `CLIENT_ORIGINS` environment variable.
