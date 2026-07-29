# Everlore Server — Schema Reference

Canonical request/response JSON shapes derived from `everlore-server/src/schemas/`, controllers, and services (`billing.service`, `auth.service.serializeUser`, `play-ws.service`, worker pub/sub).

When code and docs disagree, trust the code. Companion: [API.md](./API.md).

---

## ID conventions

| Context | Field | Format |
|---------|-------|--------|
| Mongo / many HTTP docs | `_id`, `*_id` | 24-char hex ObjectId string in JSON |
| Auth / persona API | `id` | Same ObjectId hex via `idString()` |
| Admin list items | `id` (serialized from `_id`) | Same |
| WebSocket / Redis | `instanceId`, `eventId`, `userId`, `jobId` | camelCase strings |

Example IDs:

```
userId      = "674a1b2c3d4e5f6071829301"
templateId  = "674a1b2c3d4e5f6071829302"
instanceId  = "674a1b2c3d4e5f6071829303"
eventId     = "674a1b2c3d4e5f6071829304"
personaId   = "674a1b2c3d4e5f6071829305"
characterId = "674a1b2c3d4e5f6071829306"
```

No `usr_` / `inst_` / `evt_` prefixes.

---

## Auth

### Request bodies (`user.schema.ts`)

**Register**

```json
{
  "email": "user@example.com",
  "username": "player123",
  "password": "securePassword123"
}
```

**Login**

```json
{
  "email": "user@example.com",
  "password": "securePassword123"
}
```

**Google**

```json
{
  "id_token": "eyJhbGciOiJSUzI1NiIs…"
}
```

**OTP send / verify**

```json
{ "phone": "+15551234567" }
```

```json
{ "phone": "+15551234567", "code": "123456" }
```

**Preferences** (all optional)

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

`narration_length`: `concise` | `detailed` | `verbose`  
`gender`: `male` | `female` | `non_binary`

**Admin set tier**

```json
{ "tier": "premium" }
```

`tier`: `free` | `premium` | `creator`

### Auth API shape (`serializeUser`)

Returned by register/login/google/otp verify (`user` field) and `GET /auth/me`:

```json
{
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
```

Defaults for new users match `defaultUserPreferences()`; new accounts seed `token_balance: 15000`.

> **Ink truth:** `token_balance` is **not** the Story Ink wallet. Use `GET /billing/me` → `balance`.

### Auth token response envelope

```json
{
  "token": "eyJhbGciOiJIUzI1NiIs…",
  "user": { /* serializeUser */ }
}
```

JWT payload (`toJwtPayload`): `{ id, email, username, tier }`.

### Account delete

```json
{ "deleted": true }
```

---

## Billing

Source: `billing.service.ts` → `catalog()`, `wallet()`, Play verify/simulate, admin grant.

### Catalog — `GET /billing/catalog`

```json
{
  "premium": {
    "tier": "premium",
    "monthly_ink": 3000,
    "daily_story_safety_cap": 160
  },
  "creator": {
    "tier": "creator",
    "monthly_ink": 6000,
    "daily_story_safety_cap": 320
  },
  "free": {
    "tier": "free",
    "monthly_ink": 60,
    "daily_story_safety_cap": 25
  },
  "welcome_ink": 180,
  "costs": {
    "story_turn": 1,
    "character_autofill": 12,
    "world_autofill": 20,
    "image_preview": 45
  },
  "purchases_enabled": true,
  "simulation_enabled": false,
  "products": {
    "everlore_premium": { "kind": "subscription", "tier": "premium" },
    "everlore_creator": { "kind": "subscription", "tier": "creator" },
    "everlore_ink_100": { "kind": "consumable", "ink": 100 },
    "everlore_ink_350": { "kind": "consumable", "ink": 350 },
    "everlore_ink_900": { "kind": "consumable", "ink": 900 }
  }
}
```

Prices are **not** included (Play Console only).

### Wallet — `GET /billing/me` (and verify / simulate / admin grant responses)

```json
{
  "tier": "premium",
  "balance": 3120,
  "profile": {
    "tier": "premium",
    "monthly_ink": 3000,
    "daily_story_safety_cap": 160
  },
  "entitlement": {
    "product_id": "everlore_premium",
    "base_plan_id": null,
    "expires_at": "2026-08-30T00:00:00.000Z"
  }
}
```

| Field | Meaning |
|-------|---------|
| `tier` | Active entitlement tier, else JWT/user fallback |
| `balance` | Sum of ink ledger deltas (authoritative Story Ink) |
| `profile` | Catalog allowance row for that tier |
| `entitlement` | Active subscription row or `null` |

Welcome ink (`welcome_ink: 180`) is upserted on first wallet read (`idempotency_key: welcome:v1`).

### Verify Google — `POST /billing/google/verify`

**Request**

```json
{
  "product_id": "everlore_ink_100",
  "purchase_token": "opaque-play-token",
  "kind": "consumable"
}
```

`kind`: `subscription` | `consumable`

**Response:** wallet object above.

### Simulate purchase — `POST /billing/simulate-purchase`

**Request**

```json
{
  "product_id": "everlore_premium",
  "idempotency_key": "qa-checkout-1"
}
```

**Response:** wallet. Unavailable → HTTP `404`.

### Admin ink grant — `POST /admin/users/:userId/ink-grants`

**Request**

```json
{
  "amount": 500,
  "idempotency_key": "support-42",
  "note": "QA top-up"
}
```

**Response:** wallet.

### RTDN — `POST /billing/google/rtdn`

**Request** (Pub/Sub style)

```json
{
  "message": {
    "data": "<base64 JSON notification>"
  }
}
```

Decoded payload may include `subscriptionNotification`, `oneTimeProductNotification`, or `voidedPurchaseNotification`. Response varies (`accepted`, wallet sync outcomes, etc.).

### Internal reserve result (not a public route)

When enforcement is on, `billingService.reserve` returns:

```json
{
  "reservation_id": "reserve:story_turn:<uuid>",
  "cost": 1,
  "balance": 179
}
```

When enforcement is off: `{ reservation_id: null, cost: 0, balance: null }`. Insufficient funds → HTTP `402` `"Not enough Story Ink"` (WS surfaces the message string).

---

## Personas

### Limits

| Field | Cap |
|-------|-----|
| `name` | 60 |
| `description` | 500 |
| `other_info` | 500 |
| `age` | 13–120 or `null` |

### Create body — `POST /personas/`

```json
{
  "name": "Aria Vale",
  "gender": "female",
  "age": 24,
  "description": "Soft-spoken cartographer",
  "other_info": "Keeps a red journal"
}
```

`gender` required: `male` | `female` | `non_binary`.

### Patch body — `PATCH /personas/:id`

Same fields, all optional.

### Persona API object

```json
{
  "id": "674a1b2c3d4e5f6071829305",
  "name": "Aria Vale",
  "gender": "female",
  "age": 24,
  "description": "Soft-spoken cartographer",
  "other_info": "Keeps a red journal",
  "created_at": "2026-07-01T12:00:00.000Z",
  "updated_at": "2026-07-01T12:00:00.000Z"
}
```

### List response

```json
{
  "personas": [ /* Persona API objects */ ],
  "total": 3,
  "page": 1
}
```

### Create / update response

```json
{ "persona": { /* Persona API object */ } }
```

### Delete response

```json
{ "success": true }
```

---

## Templates

### Field limits (`FIELD_LIMITS`)

| Field | Max chars |
|-------|-----------|
| `title` | 80 |
| `description` | 600 |
| `seed_prompt` | 2500 |
| `global_lore` | 3000 |
| `opening_line` | 600 |
| `style_notes` | 500 |
| `image_prompt` | 1400 |
| protagonist `name` | 80 |
| protagonist text fields | 400 |

### Create template — `POST /templates/` {#create-template}

```json
{
  "title": "Ashen Coast",
  "description": "A rain-soaked frontier…",
  "kind": "world",
  "is_sentient": false,
  "is_nsfw_capable": false,
  "seed_prompt": "You are the chronicler of…",
  "global_lore": "…",
  "narrative_style": "literary",
  "style_notes": "…",
  "image_url": "https://…",
  "image_prompt": "…",
  "opening_line": "…",
  "protagonist": {
    "name": "Aria",
    "persona": "…",
    "appearance": "…"
  },
  "base_stats_template": {
    "resolve": {
      "default": 50,
      "min": 0,
      "max": 100,
      "description": "Willpower"
    }
  },
  "flag_definitions": {
    "met_captain": {
      "type": "boolean",
      "default": false,
      "description": "Met the harbor captain"
    }
  },
  "scene_tags": ["harbor", "night"],
  "model_preferences": {
    "logic": "…",
    "narration_nsfw": "…",
    "narration_sfw": "…",
    "summary": "…"
  },
  "max_context_memories": 20,
  "max_lore_results": 8
}
```

| Field | Notes |
|-------|--------|
| `kind` | optional `world` \| `character` |
| `is_sentient` / `is_nsfw_capable` | required booleans |
| `seed_prompt` | min 10 |
| `base_stats_template` | optional; worlds still require ≥1 stat in service |
| `flag_definitions.*.type` | `boolean` \| `integer` \| `string` |
| `scene_tags` | max 8 items, each ≤24 chars |
| `max_context_memories` | 5–50 |
| `max_lore_results` | 3–20 |

**Update** (`PUT /templates/:id`): partial of the same object.

### Image generate

**Request:** `{ "prompt": "…" }` (4–1400)  
**Response:** `{ "url": "https://…", "key": "previews/…" }`

### Autofill

**Request**

```json
{
  "target": "character",
  "brief": "A lonely lighthouse keeper…",
  "is_sentient": true,
  "is_nsfw_capable": false,
  "narrative_style": "literary"
}
```

**Response**

```json
{
  "target": "character",
  "draft": { }
}
```

(`draft` shape is autofill-service specific: world vs character field bundles.)

### Quota — `GET /templates/mine/quota`

```json
{
  "can_create": false,
  "reason": "daily_limit",
  "remaining": 0,
  "retry_after": 3600
}
```

`reason`: `null` | `"tier"` | `"daily_limit"`. `remaining` may be `null` when unlimited.

### Delete

```json
{ "deleted": true }
```

---

## Instances

### Create — `POST /instances/`

**Request**

```json
{
  "template_id": "674a1b2c3d4e5f6071829302",
  "persona_id": "674a1b2c3d4e5f6071829305"
}
```

**Response:** `{ "instance": { /* WorldInstanceDoc */ }, "template": { /* WorldTemplateDoc */ } }`

### Query params (list / realms)

```json
{
  "page": 1,
  "limit": 12,
  "include_archived": false,
  "search": "coast"
}
```

### Play status

```json
{
  "has_played": true,
  "count": 2,
  "latest_instance_id": "674a1b2c3d4e5f6071829303",
  "stories": [
    {
      "id": "674a1b2c3d4e5f6071829303",
      "last_active_at": "2026-07-30T01:00:00.000Z",
      "total_events": 40
    }
  ]
}
```

### Realms list

```json
{
  "realms": [ ],
  "total": 10,
  "page": 1
}
```

### By-template

```json
{
  "template": { },
  "stories": [
    {
      "preview": "…",
      "story_index": 2
    }
  ]
}
```

(Each story also carries instance fields + nested `template` summary.)

### Protagonist

**Request**

```json
{
  "name": "Aria",
  "identity": "A cartographer from the dunes",
  "reuse_from_instance_id": "674a…"
}
```

**Response**

```json
{
  "protagonist": {
    "id": "674a1b2c3d4e5f6071829306",
    "canonical_name": "Aria"
  }
}
```

### Reusable protagonists

```json
{
  "protagonists": [
    {
      "source_instance_id": "674a…",
      "name": "Aria",
      "identity": "…",
      "appearance": "…"
    }
  ]
}
```

### Settings patch — `PATCH /instances/:id/settings`

**Request** (all optional)

```json
{
  "narration_pov": "third",
  "mode": "story",
  "message_length": "long",
  "narrative_style_override": null,
  "narration_tone": "warm",
  "focus_character_id": null,
  "persona_id": "674a…"
}
```

**Response** — normalized:

```json
{
  "narration_pov": "third",
  "mode": "story",
  "message_length": "long",
  "narrative_style_override": null,
  "narration_tone": "warm",
  "focus_character_id": null,
  "persona_id": "674a1b2c3d4e5f6071829305"
}
```

### Archive / reset / delete

```json
{ "success": true }
```

```json
{ "reset": true }
```

```json
{ "deleted": true }
```

---

## Chronicle

### Event query

```json
{
  "page": 1,
  "limit": 30,
  "before_sequence": 100,
  "type": "narration"
}
```

### Memory query

```json
{
  "include_archived": false,
  "q": "sister",
  "type": "relationship",
  "min_importance": 3,
  "unresolved": true
}
```

### Edit event

```json
{
  "ai_response": "…",
  "player_input": "…"
}
```

### Edit memory

```json
{
  "text": "Mira is my sister.",
  "type": "relationship",
  "importance": 5
}
```

### Edit character

```json
{
  "canonical_name": "Mira",
  "role": "Harbor clerk",
  "appearance": "…",
  "persona": "…",
  "immutable_facts": ["Born in Ashen Coast"],
  "mutable_state": ["Worried about the storm"],
  "disposition_to_player": "Warm",
  "hidden_thought": "…"
}
```

### Kinship GET

```json
{
  "relations": [
    {
      /* confirmed kinship edge to self — service-shaped */
    }
  ]
}
```

### Kinship POST body

```json
{
  "character": "Mira",
  "relation": "sister",
  "correction": true,
  "replaces_relation": "friend"
}
```

**Response:** `{ "saved": true }`

### Relation candidates GET

```json
{
  "candidates": [
    {
      "id": "674a…",
      "kind": "kinship",
      "character_name": "Mira",
      "counterpart_character_name": null,
      "proposed_name": null,
      "replaces_relation": null,
      "relation": "sister",
      "evidence": "…",
      "sequence": 42
    }
  ]
}
```

`kind` may also be `identity_rename` (uses `proposed_name`).

### Relation candidate resolve

**Request**

```json
{
  "action": "accept",
  "relation": "sister"
}
```

`action`: `accept` | `reject` | `defer`

**Response (reject/defer):** `{ "resolved": true }`  
Accept may return additional character/rename fields depending on kind.

### Timeline fork

```json
{
  "name": "What if we stayed?",
  "timeline_id": "optional-id",
  "parent_timeline_id": "main",
  "make_active": true
}
```

### Set active timeline

```json
{ "timeline_id": "main" }
```

### Event time-anchor

```json
{
  "story_calendar": {
    "year": 1042,
    "month": 3,
    "day": 12,
    "era": "Third Age",
    "label": "Spring tide"
  },
  "event_time_label": "Dawn",
  "timeline_id": "main"
}
```

### Replay select

```json
{ "variant_index": 0 }
```

### Rewind

```json
{ "sequence": 12 }
```

### Track entity

```json
{
  "name": "Mira",
  "role": "Harbor clerk",
  "appearance": "…",
  "persona": "…",
  "relation_kind": "sibling_of",
  "relation_label": "sister",
  "relation_to": "player"
}
```

`relation_kind` enum: `parent_of` | `child_of` | `sibling_of` | `partner_of` | `progenitor_of` | `descendant_of` | `superior_of` | `subordinate_of` | `kin_of` | `bonded_of`.

---

## Admin list envelope

```json
{
  "total": 120,
  "page": 1,
  "limit": 50,
  "items": [
    {
      "id": "674a…",
      "username": "player123"
    }
  ]
}
```

ObjectIds become hex strings; `_id` keys become `id`.

### Overview

```json
{
  "users": 100,
  "worlds": 40,
  "published_worlds": 22,
  "world_instances": 300,
  "events": 50000,
  "memories": 80000,
  "characters": 4000
}
```

### Continuity audit status list extras

```json
{
  "total": 10,
  "page": 1,
  "limit": 50,
  "status": "unhealthy",
  "stale_days": 14,
  "items": [
    {
      "id": "…",
      "player_id": "…",
      "template_id": "…",
      "total_events": 40,
      "last_active_at": "…",
      "updated_at": "…",
      "last_continuity_audit": { }
    }
  ]
}
```

---

## WebSocket message envelopes

### Client → server (`WsMessage`)

```json
{
  "action": "chat",
  "instance_id": "674a1b2c3d4e5f6071829303",
  "event_id": "674a1b2c3d4e5f6071829304",
  "payload": {}
}
```

| `action` | Payload |
|----------|---------|
| `chat` | `{ "message": "I walk to the pier." }` |
| `continue` | `{ "advance": "day" }` optional |
| `world_action` | travel or relationship object |
| `side_chat` | `{ "character_id": "…", "message": "…" }` |
| `replay` | (uses top-level `event_id`) |
| `load_instance` | — |
| `ping` | — |

**Travel world action**

```json
{
  "kind": "travel",
  "destination": "The Market",
  "companions": ["Mira"],
  "time_advance": "hours"
}
```

**Relationship world action**

```json
{
  "kind": "relationship",
  "character": "Mira",
  "relation": "sister",
  "correction": false,
  "replaces_relation": "friend"
}
```

### Server → client (direct)

```json
{ "type": "connected", "userId": "674a…" }
```

```json
{ "type": "ack", "jobId": "…" }
```

```json
{ "type": "pong" }
```

```json
{
  "type": "instance_loaded",
  "data": {
    "instance": { },
    "template": { },
    "recentEvents": [ ],
    "memories": [ ],
    "characters": [ ],
    "operation": null,
    "eventWindow": {
      "limit": 40,
      "total": 120,
      "hasOlder": true
    }
  }
}
```

```json
{ "type": "account_deleted" }
```

### Errors

```json
{
  "type": "error",
  "code": "RATE_LIMITED",
  "retryAfter": 12
}
```

```json
{
  "type": "error",
  "code": "GENERATION_IN_PROGRESS"
}
```

```json
{
  "type": "error",
  "message": "Not enough Story Ink"
}
```

```json
{
  "type": "error",
  "code": "REPLAY_FAILED",
  "eventId": "674a…",
  "message": "…"
}
```

```json
{
  "type": "error",
  "message": "Unknown action: foo"
}
```

### Generation stream (Redis → WS)

```json
{ "type": "generation_started", "instanceId": "…" }
```

```json
{ "type": "generation_delta", "instanceId": "…", "delta": "rain…" }
```

```json
{
  "type": "generation_stream_end",
  "instanceId": "…",
  "narrative": "Full visible prose…"
}
```

```json
{
  "type": "choices_ready",
  "instanceId": "…",
  "choices": [ ]
}
```

```json
{
  "type": "generation_complete",
  "instanceId": "…",
  "event": {
    "id": "…",
    "sequence": 41,
    "player_input": "…",
    "narrative": "…",
    "scene_tag": "…",
    "emotional_tone": "…",
    "model_used": "…",
    "choices": [ ],
    "milestone": null,
    "present_characters": [ ],
    "trackable_mentions": [ ],
    "time_advanced": null,
    "time_anchor": { },
    "location_anchor": { },
    "fate_thread": null,
    "event_type": "narration",
    "state_diff": {
      "world_state": { },
      "active_flags": { }
    }
  }
}
```

```json
{ "type": "generation_reset", "instanceId": "…" }
```

```json
{
  "type": "generation_retrying",
  "instanceId": "…",
  "attempt": 1,
  "maxAttempts": 3
}
```

```json
{
  "type": "generation_failed",
  "instanceId": "…",
  "message": "The world could not respond. Please try again."
}
```

```json
{
  "type": "milestone_unlocked",
  "instanceId": "…",
  "milestone": { "label": "…", "sequence": 41 }
}
```

### Side chat

```json
{
  "type": "side_chat_delta",
  "instanceId": "…",
  "characterId": "…",
  "delta": "…"
}
```

```json
{
  "type": "side_chat_complete",
  "instanceId": "…",
  "character": { "id": "…", "name": "Mira" },
  "reachability": { "mode": "…", "reason": "…" },
  "event": {
    "id": "…",
    "sequence": 42,
    "player_input": "…",
    "narrative": "…",
    "model_used": "…",
    "created_at": "2026-07-30T00:00:00.000Z"
  }
}
```

```json
{
  "type": "side_chat_error",
  "instanceId": "…",
  "characterId": "…",
  "message": "…"
}
```

### Replay

```json
{
  "type": "replay_delta",
  "instanceId": "…",
  "eventId": "…",
  "delta": "…"
}
```

```json
{
  "type": "replay_complete",
  "instanceId": "…",
  "eventId": "…",
  "narrative": "…",
  "selected_index": 0,
  "choices": [ ],
  "present_characters": [ ],
  "trackable_mentions": [ ],
  "instance_state": null,
  "variants": [
    {
      "id": "…",
      "narrative": "…",
      "model_used": "…",
      "created_at": "…",
      "choices": [ ],
      "present_characters": [ ],
      "trackable_mentions": [ ]
    }
  ]
}
```

### Projection updates

```json
{
  "type": "character_codex_updated",
  "instanceId": "…",
  "characters": [ ]
}
```

(Exact payload fields may include full codex list; clients should treat the event as authoritative refresh.)

```json
{
  "type": "world_projection_updated",
  "instance_id": "…",
  "scopes": ["bonds", "threads", "recap", "places", "calendar", "codex", "presence"],
  "source": "replay"
}
```

---

## Related docs

- [API.md](./API.md) — endpoints and protocol  
- [BILLING.md](./BILLING.md) — Play / RTDN / enforcement  
- [DATA_MODEL.md](./DATA_MODEL.md) — Mongo collections  
- [CONFIGURATION.md](./CONFIGURATION.md) · [SECURITY.md](./SECURITY.md)  
