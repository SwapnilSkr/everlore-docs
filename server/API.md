# Everlore Server — API Documentation

> **Schema reference:** [SCHEMAS.md](./SCHEMAS.md) — request/response JSON shapes.  
> **Billing product detail:** [BILLING.md](./BILLING.md) · **Env & flags:** [CONFIGURATION.md](./CONFIGURATION.md) · **Auth & admin hardening:** [SECURITY.md](./SECURITY.md)  
> IDs are 24-char MongoDB **ObjectId** hex strings. There are no `usr_` / `inst_` prefixes.  
> Source of truth: `everlore-server/src/routes/*.routes.ts`, controllers, schemas, `play-ws.service.ts`, and `worker/index.ts`.

## Base URL

```
http://localhost:{PORT}
```

`PORT` defaults from server env (see [CONFIGURATION.md](./CONFIGURATION.md)).

## Authentication

| Surface | Auth |
|---------|------|
| Most `/auth/*` login/register/OTP routes | Public |
| `GET /auth/me`, `PUT /auth/preferences`, `DELETE /auth/account` | Bearer JWT |
| `/templates`, `/instances`, `/personas`, `/chronicle`, `/billing` (except noted) | Bearer JWT |
| `GET /templates/` and `GET /templates/:id` (published) | Optional Bearer (anonymous can browse published) |
| `/admin/*` | HTTP Basic (`ADMIN_USERNAME` / `ADMIN_PASSWORD`) |
| `POST /billing/google/rtdn` | Google Play RTDN bearer (not JWT) |
| `WS /ws/play` | JWT via query `?token=` |

```
Authorization: Bearer {jwt_token}
```

JWTs are issued by `/auth/register`, `/auth/login`, `/auth/google`, and `/auth/otp/verify`. Payload carries `id`, `email`, `username`, `tier`. On HTTP requests the auth plugin **refreshes `tier` from the live user document** so entitlement changes apply without waiting for re-login. Admin tier patches note that existing JWTs may still show the old tier until the next sign-in for WebSocket/`verifyWsToken` paths that do not re-fetch tier.

### Story Ink vs `token_balance`

- **Authoritative Story Ink wallet:** `GET /billing/me` → `balance` (ledger-backed).
- Auth responses still include `user.token_balance` from `serializeUser` for legacy/compat. **Do not treat `token_balance` as the Story Ink balance.** Use `/billing/me`.

## Content Types

```
Content-Type: application/json
```

Optional for billable forge calls: `X-Idempotency-Key` (image generate / autofill).

## Request handling

Routes in `src/routes/` → controllers in `src/controllers/` → services in `src/services/`. Workers publish play events over Redis `user:{userId}:events`, relayed to open WebSockets.

---

## Health

### GET `/`

```
"Everlore API"
```

### GET `/health`

```json
{
  "ok": true,
  "timestamp": "2026-07-30T00:00:00.000Z"
}
```

No auth.

---

## Auth (`/auth`)

### POST `/auth/register`

**Body:** `{ email, username, password }`

| Field | Rules |
|-------|--------|
| `email` | Valid email |
| `username` | 3–30 chars, `^[a-zA-Z0-9_]+$` |
| `password` | 8–128 chars |

**Response:** `{ token, user }` where `user` is `serializeUser` (see [SCHEMAS.md](./SCHEMAS.md)).

**Errors:** `409` email already registered / username taken.

---

### POST `/auth/login`

**Body:** `{ email, password }`

**Response:** `{ token, user }`

**Rate limit:** `auth_attempt` per email → `429` “Too many login attempts…”

**Errors:** `401` Invalid credentials.

---

### POST `/auth/google`

**Body:** `{ id_token }`

Verifies Google ID token; creates user if needed (`providers` includes `google`).

**Response:** `{ token, user }`

---

### POST `/auth/otp/send`

**Body:** `{ phone }` — E.164 `^\+[1-9]\d{7,14}$`

**Response:** `{ success: true, mockCode: "123456" }`

`mockCode` is always returned by the current controller (local/mock convenience). Rate limit `otp_send`.

---

### POST `/auth/otp/verify`

**Body:** `{ phone, code }` (`code` 4–10 chars)

**Response:** `{ token, user }`

Rate limit `otp_verify`. Creates phone users when new.

---

### GET `/auth/me`

**Auth required.** Returns `serializeUser` for the current user.

---

### PUT `/auth/preferences`

**Auth required.**

**Body** (all optional):

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

| Field | Values |
|-------|--------|
| `narration_length` | `concise` \| `detailed` \| `verbose` |
| `gender` | `male` \| `female` \| `non_binary` |
| `player_name` | 2–40 chars |

**Response:** `{ "success": true }`

---

### DELETE `/auth/account`

**Auth required.** Permanently deletes the account (all owned instances, created templates, user doc), then disconnects play WebSockets with `{ type: "account_deleted" }`.

**Response:** `{ "deleted": true }`

---

## Billing (`/billing`)

Full product / Play / RTDN setup: [BILLING.md](./BILLING.md). Costs and catalog live in `billing.service.ts` (`BILLING_CATALOG`, `PLAY_PRODUCTS`).

### GET `/billing/catalog`

Public (auth plugin mounted but handler ignores user). Returns entitlement metadata — **not store prices**.

Includes:

- Tier profiles: `free` / `premium` / `creator` (`monthly_ink`, `daily_story_safety_cap`)
- `welcome_ink`
- `costs`: `story_turn`, `character_autofill`, `world_autofill`, `image_preview`
- `purchases_enabled`, `simulation_enabled`
- `products`: Play product id → `{ kind, tier? }` or `{ kind, ink? }`

See [SCHEMAS.md](./SCHEMAS.md#billing-catalog).

---

### GET `/billing/me`

**Auth required.** Authoritative Story Ink wallet.

Ensures welcome ink grant, then returns:

```json
{
  "tier": "free",
  "balance": 180,
  "profile": {
    "tier": "free",
    "monthly_ink": 60,
    "daily_story_safety_cap": 25
  },
  "entitlement": null
}
```

When a subscription entitlement is active, `entitlement` is `{ product_id, base_plan_id, expires_at }`.

---

### POST `/billing/google/verify`

**Auth required.**

**Body:**

```json
{
  "product_id": "everlore_ink_100",
  "purchase_token": "…",
  "kind": "subscription"
}
```

`kind`: `subscription` \| `consumable`.

**Response:** Updated wallet (same shape as `/billing/me`).

---

### POST `/billing/simulate-purchase`

**Auth required.** QA-only. Returns `404` when simulation is off (`BILLING_SIMULATION_ENABLED` and not production).

**Body:**

```json
{
  "product_id": "everlore_premium",
  "idempotency_key": "qa-checkout-1"
}
```

**Response:** Updated wallet.

---

### POST `/billing/google/rtdn`

**Not JWT.** Google Play Real-Time Developer Notification webhook. Authorization verified via `googlePlayService.verifyRtdnBearer`.

**Body:** Pub/Sub envelope `{ message?: { data?: string } }` — `data` is base64 JSON.

Handles subscription / one-time / voided purchase notifications; syncs or voids purchases.

---

## Personas (`/personas`)

All require Bearer JWT. Limits: name 60, description/other_info 500, age 13–120.

### GET `/personas/`

**Query:** `page?`, `limit?` (max 50), `search?` (max 100)

**Response:** `{ personas, total, page }` (persona API shape).

### POST `/personas/`

**Body:** `{ name, gender, age?, description?, other_info? }`

`gender`: `male` \| `female` \| `non_binary`.

**Response:** `{ persona }`

### PATCH `/personas/:id`

Partial update of the same fields.

**Response:** `{ persona }`

### DELETE `/personas/:id`

Clears `persona_id` / snapshot on linked instances and busts session cache.

**Response:** `{ "success": true }`

---

## Templates (`/templates`)

Auth plugin on all routes. Create / update / publish / delete / mine / quota / image / autofill require a signed-in user. List published and get-by-id allow anonymous for **published** templates; unpublished drafts require owner.

### GET `/templates/`

**Query:** `page?`, `limit?`, `search?`

Published template list (paginated).

### GET `/templates/mine/list`

**Auth required.** Creator’s templates.

**Query:** `page?` (≥1), `limit?` (1–50), `search?` (max 100)

### GET `/templates/mine/quota`

**Auth required.** Daily creation budget **before** spending money on previews.

```json
{
  "can_create": true,
  "reason": null,
  "remaining": 3,
  "retry_after": null
}
```

Non-creator/premium: `{ can_create: false, reason: "tier", remaining: 0, retry_after: null }`. Over daily limit: `reason: "daily_limit"`.

### GET `/templates/:id`

Published: anyone. Unpublished: owner only (`401`/`403` otherwise).

### POST `/templates/image/generate`

**Auth:** premium or creator. Rate limit `image_generate`. Reserves `image_preview` Ink.

**Body:** `{ "prompt": "…" }` (4–1400 chars)

**Headers:** optional `X-Idempotency-Key`

**Response:** `{ url, key }`

**Errors:** `403` tier, `429` rate, `402` Not enough Story Ink (when enforcement on).

### POST `/templates/autofill`

**Auth:** premium or creator. Rate limit `autofill`. Reserves `character_autofill` or `world_autofill`.

**Body:**

```json
{
  "target": "world",
  "brief": "…",
  "is_sentient": false,
  "is_nsfw_capable": false,
  "narrative_style": "literary"
}
```

**Response:** `{ target, draft }`

### POST `/templates/`

**Auth:** premium or creator. Consumes daily `template_create` quota only after successful create.

**Body:** `CreateTemplateBody` — see [SCHEMAS.md](./SCHEMAS.md#create-template).

### PUT `/templates/:id`

Owner update. Body: partial create fields.

### POST `/templates/:id/publish`

Owner publish.

### DELETE `/templates/:id`

Owner hard-delete (cascades playthroughs). **Response:** `{ deleted: true }`

---

## Instances (`/instances`)

All require Bearer JWT.

### GET `/instances/`

**Query:** `include_archived?`, plus schema allows `page`/`limit`/`search` (list currently uses `include_archived` only).

Returns instance rows with embedded template summary.

### GET `/instances/realms`

One card per template (grouped playthroughs).

**Query:** `page?`, `limit?` (capped ~30), `include_archived?`, `search?`

**Response:** `{ realms, total, page }`

### GET `/instances/play-status/:templateId`

Fast “have I played this world?” check.

```json
{
  "has_played": true,
  "count": 2,
  "latest_instance_id": "674a…",
  "stories": [
    { "id": "674a…", "last_active_at": "…", "total_events": 40 }
  ]
}
```

### GET `/instances/by-template/:templateId`

All active stories for one template with previews: `{ template, stories }`.

### GET `/instances/:id/reusable-protagonists`

Protagonists from sibling playthroughs of the same template (empty for sentient worlds).

**Response:** `{ protagonists: [{ source_instance_id, name, identity, appearance }] }`

### GET `/instances/:id`

Owned instance document or `404`.

### POST `/instances/`

**Body:** `{ "template_id": "…", "persona_id": "…" }` (`persona_id` optional)

**Response:** `{ instance, template }`

Enforces tier instance limits; creators may start unpublished own templates.

### POST `/instances/:id/archive`

**Response:** `{ success: true }`

### POST `/instances/:id/protagonist`

GM onboarding for non-sentient worlds.

**Body:**

```json
{
  "name": "Aria",
  "identity": "…",
  "reuse_from_instance_id": "674a…"
}
```

**Response:** `{ protagonist: { id, canonical_name } | null }`

### POST `/instances/:id/reset`

Purges story data; preserves protagonist identity. **Response:** `{ reset: true }`

### DELETE `/instances/:id`

Hard delete. **Response:** `{ deleted: true }`

### PATCH `/instances/:id/settings`

**Body** (all optional):

```json
{
  "narration_pov": "first",
  "mode": "story",
  "message_length": "medium",
  "narrative_style_override": null,
  "narration_tone": "warm",
  "focus_character_id": null,
  "persona_id": null
}
```

| Field | Values |
|-------|--------|
| `narration_pov` | `first` \| `third` |
| `message_length` | `short` \| `medium` \| `long` |
| `focus_character_id` / `persona_id` | ObjectId string or `null` |

**Response:** Normalized settings object (same keys).

---

## Chronicle (`/chronicle`)

All require Bearer JWT and ownership of the instance/resources.

### Feed & memory

| Method | Path | Notes |
|--------|------|--------|
| GET | `/chronicle/events/:instanceId` | Query: `page`, `limit`, `before_sequence`, `type` |
| GET | `/chronicle/memories/:instanceId` | Query: `include_archived`, `q`, `type`, `min_importance`, `unresolved` |
| GET | `/chronicle/calendar/:instanceId` | Story calendar |
| GET | `/chronicle/recap/:instanceId` | Recap projection |
| GET | `/chronicle/threads/:instanceId` | Open threads |
| GET | `/chronicle/relationships/:instanceId` | Relationship sheet |
| GET | `/chronicle/relationships/:instanceId/:characterId/memories` | Per-character memories |
| GET | `/chronicle/locations/:instanceId` | Places atlas |
| GET | `/chronicle/locations/:instanceId/:locationEntityId` | Location journal |

### Kinship & relation review

#### GET `/chronicle/kinship/:instanceId`

**Response:** `{ relations: […] }` — confirmed kinship edges to self (player or protagonist depending on sentient template).

#### POST `/chronicle/kinship/:instanceId`

Player authorial kinship write (not a chat turn).

**Body:**

```json
{
  "character": "Mira",
  "relation": "sister",
  "correction": false,
  "replaces_relation": "friend"
}
```

| Field | Rules |
|-------|--------|
| `character` | 2–80 chars |
| `relation` | 2–40 chars (surface label → ontology kind) |
| `correction` | optional bool; if true, `replaces_relation` required |
| `replaces_relation` | optional, max 40 |

**Response:** `{ saved: true }`  
**Errors:** `400` invalid, `409` player not established / could not resolve.

#### GET `/chronicle/relation-candidates/:instanceId`

**Response:** `{ candidates: […] }` — open review items (`kind`, names, `relation`, `evidence`, `sequence`, …).

#### POST `/chronicle/relation-candidate/:candidateId/resolve`

**Body:**

```json
{
  "action": "accept",
  "relation": "sister"
}
```

`action`: `accept` \| `reject` \| `defer`. Optional `relation` on accept.

**Response:** `{ resolved: true }` (or accept path may return richer character update depending on candidate kind, e.g. `identity_rename`).

### Side chats

| Method | Path |
|--------|------|
| GET | `/chronicle/side-chats/:instanceId` |
| GET | `/chronicle/side-chats/:instanceId/:characterId` |
| GET | `/chronicle/side-chats/:instanceId/:characterId/reachability` |

### Timeline / calendar edits

#### POST `/chronicle/calendar/:instanceId/timeline`

**Body:** `{ name, timeline_id?, parent_timeline_id?, make_active? }`

#### PUT `/chronicle/calendar/:instanceId/timeline/active`

**Body:** `{ timeline_id }`

#### PUT `/chronicle/calendar/event/:eventId/time-anchor`

**Body:** optional `story_calendar` (`year`, `month`, `day`, `era`, `label`), `event_time_label`, `timeline_id`.

### Edits

#### PUT `/chronicle/memory/:memoryId`

**Body:** `{ text, type?, importance? }` — types: `relationship` \| `promise` \| `lore` \| `observation` \| `emotion` \| `secret`; importance 1–5.

#### DELETE `/chronicle/memory/:memoryId`

#### PUT `/chronicle/event/:eventId`

**Body:** `{ ai_response?, player_input? }` (max 10000 / 4000)

#### PUT `/chronicle/character/:characterId`

**Body** (all optional): `canonical_name`, `role`, `appearance`, `persona`, `immutable_facts[]`, `mutable_state[]`, `disposition_to_player`, `hidden_thought`.

### Replay / rewind / track

#### POST `/chronicle/replay/:eventId`

HTTP replay (also available over WS `action: "replay"`).

#### POST `/chronicle/replay/select/:eventId`

**Body:** `{ "variant_index": 0 }`

#### POST `/chronicle/rewind/:instanceId`

**Body:** `{ "sequence": 12 }` (`sequence` ≥ 1)

Takes the shared generation lock; `409` if another story op is in progress.

#### POST `/chronicle/track/:instanceId`

Promote / correct a character into the codex (+ optional kinship).

**Body:**

```json
{
  "name": "Mira",
  "role": "…",
  "appearance": "…",
  "persona": "…",
  "relation_kind": "sibling_of",
  "relation_label": "sister",
  "relation_to": "player"
}
```

`relation_kind` enum: `parent_of`, `child_of`, `sibling_of`, `partner_of`, `progenitor_of`, `descendant_of`, `superior_of`, `subordinate_of`, `kin_of`, `bonded_of`.

Publishes `character_codex_updated` / `world_projection_updated` over WS when successful.

---

## Admin (`/admin`)

**Auth:** HTTP Basic only. If `ADMIN_USERNAME` / `ADMIN_PASSWORD` unset → `503`. Invalid → `401` + `WWW-Authenticate: Basic realm="Everlore Admin"`.

List endpoints share pagination: `page` (default 1), `limit` (default 50, max 200). Many accept `search`. Patch bodies are open `Record<string, any>` (service whitelists/cleans fields). List responses: `{ total, page, limit, items }` with `_id` → `id`.

### Overview

| Method | Path | Response |
|--------|------|----------|
| GET | `/admin/overview` | Counts: users, worlds, published_worlds, world_instances, events, memories, characters |

### Users

| Method | Path | Notes |
|--------|------|--------|
| GET | `/admin/users` | Query: `page`, `limit`, `search` |
| GET | `/admin/users/:userId` | User + counts |
| PATCH | `/admin/users/:userId` | Arbitrary safe fields |
| PATCH | `/admin/users/:userId/tier` | Body `{ tier: "free"\|"premium"\|"creator" }`; response notes JWT lag |
| POST | `/admin/users/:userId/ink-grants` | See below |
| DELETE | `/admin/users/:userId` | Full account purge |

#### POST `/admin/users/:userId/ink-grants`

Support / QA Story Ink grant (independent of Play / enforcement switch).

**Body:**

```json
{
  "amount": 500,
  "idempotency_key": "support-ticket-42",
  "note": "QA top-up"
}
```

| Field | Rules |
|-------|--------|
| `amount` | 1–1_000_000 |
| `idempotency_key` | 1–120 chars |
| `note` | optional, max 240 |

**Response:** Wallet shape (same as `/billing/me`).

### Worlds (templates)

| Method | Path | Query extras |
|--------|------|----------------|
| GET | `/admin/worlds` | `search`, `creator_id`, `published` |
| GET | `/admin/worlds/:worldId` | + counts |
| PATCH | `/admin/worlds/:worldId` | |
| DELETE | `/admin/worlds/:worldId` | Cascading template delete |

### Instances

| Method | Path | Notes |
|--------|------|--------|
| GET | `/admin/instances` | `player_id`, `template_id`, `archived` |
| GET | `/admin/instances/continuity-audits` | `status`: `all`\|`healthy`\|`unhealthy`\|`missing`\|`stale`; `stale_days` 1–365 |
| GET | `/admin/instances/:instanceId` | + counts |
| GET | `/admin/instances/:instanceId/continuity-audit` | Full continuity audit |
| PATCH | `/admin/instances/:instanceId` | |
| DELETE | `/admin/instances/:instanceId` | |

### Events / memories / characters / logs

| Method | Path |
|--------|------|
| GET | `/admin/events` (`instance_id`, `player_id`) |
| GET | `/admin/events/:eventId/projections` |
| PATCH | `/admin/events/:eventId` |
| DELETE | `/admin/events/:eventId` |
| GET | `/admin/memories` |
| PATCH | `/admin/memories/:memoryId` |
| DELETE | `/admin/memories/:memoryId` |
| GET | `/admin/characters` (`instance_id`) |
| PATCH | `/admin/characters/:characterId` |
| DELETE | `/admin/characters/:characterId` |
| GET | `/admin/generation-logs` (`instance_id`, `player_id`) |

---

## WebSocket — Play (`/ws/play`)

### Connect

```
ws://host/ws/play?token={jwt}
```

On success: `{ "type": "connected", "userId": "…" }`  
Missing/invalid token: `{ type: "error", message }` then close.

Subscribe channel: Redis `user:{userId}:events` (relayed to all sockets for that user).

### Client → server frames

Validated by `WsMessage` (`src/schemas/ws-message.schema.ts`):

```json
{
  "action": "chat",
  "instance_id": "674a…",
  "event_id": "674a…",
  "payload": {}
}
```

| `action` | Required fields | Payload | Billable |
|----------|-----------------|---------|----------|
| `chat` | `instance_id` | `{ message }` 1–4000 | `story_turn` |
| `continue` | `instance_id` | optional `{ advance: "hours"\|"day"\|"days"\|"season" }` | `story_turn` |
| `world_action` | `instance_id` | travel or relationship (below) | `story_turn` |
| `side_chat` | `instance_id` | `{ character_id, message }` | `story_turn` |
| `replay` | `instance_id`, `event_id` | — | `story_turn` |
| `load_instance` | `instance_id` | — | no |
| `ping` | — | — | no → `{ type: "pong" }` |

#### `world_action` payload

**Travel:**

```json
{
  "kind": "travel",
  "destination": "The Market",
  "companions": ["Mira"],
  "time_advance": "day"
}
```

**Relationship:**

```json
{
  "kind": "relationship",
  "character": "Mira",
  "relation": "sister",
  "correction": false,
  "replaces_relation": "friend"
}
```

If `correction: true`, `replaces_relation` is required. Relation labels are a fixed allow-list (see `WORLD_ACTION_RELATIONS` in code).

### Immediate server replies (socket handler)

| Type | When |
|------|------|
| `ack` | Job queued — `{ type: "ack", jobId }` |
| `instance_loaded` | `{ type: "instance_loaded", data }` — instance, template, recentEvents, memories, characters, operation, eventWindow |
| `pong` | ping |
| `error` | See codes below |
| `account_deleted` | Account deleted while connected |

#### Error codes / messages

| Shape | Meaning |
|-------|---------|
| `{ type: "error", code: "RATE_LIMITED", retryAfter }` | Chat rate limit |
| `{ type: "error", code: "GENERATION_IN_PROGRESS" }` | Per-instance generation lock held |
| `{ type: "error", message: "Not enough Story Ink" }` | Billing reserve failed (`402` path → message string) when `BILLING_ENFORCEMENT_ENABLED` |
| `{ type: "error", message }` | Validation / auth / unknown action / other failures |
| `{ type: "error", code: "REPLAY_FAILED", eventId, message }` | Replay processor failure (via Redis) |

On dispatch failure after reserve, Ink reservation is **released** and the lock cleared.

### Worker → client events (via Redis relay)

Published by generation / side-chat / replay processors and `worker/index.ts` failure handling:

| `type` | Role |
|--------|------|
| `generation_started` | Turn began |
| `generation_delta` | Streaming prose chunk `{ instanceId, delta }` |
| `generation_stream_end` | Visible prose finished `{ instanceId, narrative }` |
| `choices_ready` | Tap choices `{ instanceId, choices }` |
| `generation_complete` | Canonical event payload (id, sequence, narrative, choices, anchors, state_diff, …) |
| `generation_reset` | Clear provisional / failed stream UI before retry or failure |
| `generation_retrying` | Intermediate failure: `{ instanceId, attempt, maxAttempts }` — more attempts remain |
| `generation_failed` | Final failure (no usable prose): `{ instanceId, message }` — Ink released; lock cleared |
| `milestone_unlocked` | `{ instanceId, milestone }` |
| `side_chat_delta` / `side_chat_complete` / `side_chat_error` | Side chat stream lifecycle |
| `replay_delta` / `replay_complete` | Replay stream + variants |
| `character_codex_updated` | Codex projection refresh |
| `world_projection_updated` | `{ instance_id, scopes, source }` |
| `error` | Including `REPLAY_FAILED` |

**Retry semantics (`worker/index.ts`):**

1. Non-final failure, no visible stream → `generation_reset` then `generation_retrying`.
2. Final failure, no visible stream → release Ink reservation, `generation_reset`, `generation_failed`, clear lock, DLQ.
3. Failure after visible stream → do **not** overwrite the player’s prose with a failure bubble; settle reservation and clear lock (tail failure logged/DLQ as appropriate).

Successful generation completion **settles** the Ink reservation (charge sticks).

---

## Related docs

- [SCHEMAS.md](./SCHEMAS.md) — JSON shapes  
- [BILLING.md](./BILLING.md) — Play products, RTDN, simulation, enforcement flags  
- [CONFIGURATION.md](./CONFIGURATION.md) — env vars  
- [SECURITY.md](./SECURITY.md) — JWT, admin Basic, secrets  
- [SERVICES.md](./SERVICES.md) — service map  
- [WORKERS.md](./WORKERS.md) — queues & processors  
