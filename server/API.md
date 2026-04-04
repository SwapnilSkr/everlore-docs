# Everlore Server - API Documentation

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
    "id": "usr_a1b2c3d4e5f6",
    "email": "user@example.com",
    "username": "player123",
    "tier": "free",
    "preferences": {
      "nsfw_enabled": false,
      "preferred_model": "gpt-4o",
      "theme": "dark",
      "narration_length": "detailed",
      "auto_memory_curation": true
    }
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

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "usr_a1b2c3d4e5f6",
    "email": "",
    "phone": "+15551234567",
    "username": "15551234567_lmn123",
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

**Rate Limiting:** 10 requests per 10 minutes per phone number

**User Creation:**
- If the phone number is new, a user is auto-created with `providers: ['phone']`
- Existing users are updated to include the `phone` provider

---

#### GET /auth/me

Get current user profile.

**Headers:** `Authorization: Bearer {token}`

**Response:**
```json
{
  "id": "usr_a1b2c3d4e5f6",
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
```

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
  "auto_memory_curation": false
}
```

**Validation:**
- `narration_length`: Must be 'concise', 'detailed', or 'verbose'

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
      "_id": "tpl_g7h8i9j0k1l2",
      "creator_id": "usr_xxx",
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
      "_id": "inst_m3n4o5p6q7r8",
      "template_id": "tpl_g7h8i9j0k1l2",
      "template_version": 1,
      "player_id": "usr_a1b2c3d4e5f6",
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
      "meta": {
        "total_events": 42,
        "total_memories": 15,
        "total_tokens_consumed": 150000,
        "last_active_at": "2025-01-15T10:30:00Z",
        "is_archived": false
      },
      "template": {
        "_id": "tpl_g7h8i9j0k1l2",
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
  "template_id": "tpl_g7h8i9j0k1l2"
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

#### GET /chronicle/events/:instanceId

Get event history for an instance.

**Query Parameters:**
- `page` (number, optional): Default 1
- `limit` (number, optional): Default 50
- `type` (string, optional): Filter by event type

**Response:**
```json
{
  "events": [
    {
      "_id": "evt_s9t0u1v2w3x4",
      "instance_id": "inst_m3n4o5p6q7r8",
      "player_id": "usr_a1b2c3d4e5f6",
      "sequence": 42,
      "type": "narration",
      "scene_tag": "exploration",
      "data": {
        "player_input": "I search the room for clues",
        "ai_response": "You find a datapad...",
        "state_mutations": {},
        "flag_mutations": {},
        "model_used": "gpt-4o",
        "tokens_in": 2500,
        "tokens_out": 400
      },
      "is_user_edited": false,
      "edit_history": [],
      "created_at": "2025-01-15T10:30:00Z"
    }
  ],
  "total": 42,
  "page": 1
}
```

**Note:** Events returned in chronological order (oldest first).

---

#### GET /chronicle/memories/:instanceId

Get memories for an instance.

**Query Parameters:**
- `include_archived` (boolean, optional): Include archived memories

**Response:** Array of memory objects

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

**Response:**
```json
{ "success": true }
```

---

## WebSocket Protocol

### Connection

```
WS /ws/play?token={jwt_token}
```

Token must be provided as query parameter (not header).

### Authentication Flow

1. Client connects with token
2. Server validates JWT
3. Server subscribes to `user:{userId}:events` Redis channel
4. Server sends: `{ "type": "connected", "userId": "usr_xxx" }`

### Client → Server Messages

All messages follow this structure:

```json
{
  "action": "chat" | "ping" | "load_instance",
  "instance_id": "inst_xxx",  // Required for most actions
  "payload": { ... }          // Action-specific data
}
```

#### Action: `chat`

Send a message to the AI.

**Request:**
```json
{
  "action": "chat",
  "instance_id": "inst_m3n4o5p6q7r8",
  "payload": {
    "message": "I examine the strange device"
  }
}
```

**Validation:**
- Message: 1-4000 characters
- Rate limit: 10 per minute

**Immediate Response:**
```json
{ "type": "ack", "jobId": "bull:generation:123" }
```

**Error Cases:**
```json
{ "type": "error", "code": "RATE_LIMITED", "retryAfter": 45 }
{ "type": "error", "code": "GENERATION_IN_PROGRESS" }
{ "type": "error", "message": "Invalid message" }
```

**Async Response** (when generation completes):
```json
{
  "type": "generation_complete",
  "instanceId": "inst_m3n4o5p6q7r8",
  "event": {
    "id": "evt_s9t0u1v2w3x4",
    "sequence": 43,
    "narrative": "The device pulses with an eerie blue light...",
    "scene_tag": "exploration",
    "emotional_tone": "curious",
    "state_diff": {
      "world_state": { "health": 85, "sanity": 90 },
      "active_flags": { "has_weapon": true, "doors_unlocked": 3 }
    }
  }
}
```

---

#### Action: `load_instance`

Load full instance state.

**Request:**
```json
{
  "action": "load_instance",
  "instance_id": "inst_m3n4o5p6q7r8"
}
```

**Response:**
```json
{
  "type": "instance_loaded",
  "data": {
    "instance": { ... },
    "template": { ... },
    "recentEvents": [ ... ],
    "memories": [ ... ]
  }
}
```

---

#### Action: `ping`

Keep connection alive.

**Request:**
```json
{
  "action": "ping"
}
```

**Response:**
```json
{ "type": "pong" }
```

---

### Server → Client Events

Events are pushed via Redis pub/sub and sent to all connected WebSockets for the user.

#### `generation_complete`

AI response is ready.

```json
{
  "type": "generation_complete",
  "instanceId": "inst_xxx",
  "event": {
    "id": "evt_xxx",
    "sequence": 43,
    "narrative": "...",
    "scene_tag": "dialogue",
    "emotional_tone": "warm",
    "state_diff": {
      "world_state": { ... },
      "active_flags": { ... }
    }
  }
}
```

---

#### `generation_failed`

Generation job failed permanently (after retries).

```json
{
  "type": "generation_failed",
  "instanceId": "inst_xxx",
  "message": "The world could not respond. Please try again."
}
```

---

#### `memories_curated`

New memories extracted from conversation.

```json
{
  "type": "memories_curated",
  "instanceId": "inst_xxx",
  "memories": [
    {
      "id": "mem_xxx",
      "text": "The player showed kindness to the wounded guard",
      "type": "observation",
      "importance": 3
    }
  ]
}
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
