# Everlore Server — Billing & Story Ink

Story Ink is the spendable currency for narration turns, autofill, and image previews. The ledger in Mongo (`ink_ledger`) is authoritative. Google Play is the storefront for subscriptions and Ink packs.

**Source:** `everlore-server/src/services/billing.service.ts`, `google-play.service.ts`, `routes/billing.routes.ts`, `BILLING_PLAY_SETUP.md`  
**Client:** Flutter `/membership` → `BillingScreen`  
**Related:** [API.md](./API.md) · [CONFIGURATION.md](./CONFIGURATION.md) · [SECURITY.md](./SECURITY.md) · [DATA_MODEL.md](./DATA_MODEL.md)

---

## Safe-by-default

| Flag | Default | Behavior |
|------|---------|----------|
| `BILLING_ENFORCEMENT_ENABLED` | `false` | When `false`, `reserve()` returns a no-op (`reservation_id: null`, `cost: 0`) and play is not blocked for balance. In production, enforcement also turns on automatically once Google Play credentials are configured. |
| `BILLING_SIMULATION_ENABLED` | `false` | Enables `POST /billing/simulate-purchase` for local/QA |

**Hard rule:** simulation is refused when `NODE_ENV=production`, even if the flag is set. Never enable simulation on a public production app.

Enable enforcement manually in non-production only after matching test products are live. In
production, the server also enables it automatically when the Play package name and service
account are configured (an explicit `BILLING_ENFORCEMENT_ENABLED=true` remains recommended).

---

## Catalog (server metadata — not prices)

Prices live only in Google Play Console. The API returns entitlement/allowance metadata via `GET /billing/catalog`.

### Tiers

| Tier | Monthly Ink allowance (catalog profile) | Daily story safety cap |
|------|------------------------------------------|------------------------|
| `free` | 60 | 25 |
| `premium` | 3000 | 160 |
| `creator` | 6000 | 320 |

### Welcome grant

Every wallet path calls `ensureWelcomeInk`: idempotent ledger row `welcome:v1` for **+180** Ink (`BILLING_CATALOG.welcome_ink`).

### Action costs (`BILLING_CATALOG.costs`)

| Action | Ink |
|--------|-----|
| `story_turn` | 1 |
| `character_autofill` | 12 |
| `world_autofill` | 20 |
| `image_preview` | 45 |

Story turns reserve Ink in `play-ws.service.ts` before dispatching BullMQ jobs. Workers **settle** on success (and after visible-stream failure that must not refund a seen scene) or **release** on final failure before visible stream.

When enforcement is active, story turns also have a server-side daily safety cap from the
player's tier profile (`25` free / `160` premium / `320` creator). The cap counts
reservations attempted during the current UTC day, including reservations later released,
because those attempts still consume provider capacity. Reconnecting with the same request
id remains idempotent and does not consume another daily slot.

---

## Google Play products

Create these **exact** product IDs in Play Console:

| Type | Product IDs |
|------|-------------|
| Subscription | `everlore_premium`, `everlore_creator` |
| Consumable | `everlore_ink_100`, `everlore_ink_350`, `everlore_ink_900` |

Give each subscription an active monthly base plan. Configure USD anchor + country overrides (e.g. India INR). The mobile app reads localized, tax-inclusive price from Play — never hardcode prices in the app or API.

Consumable grants map to +100 / +350 / +900 Ink respectively.

---

## Server verification setup

1. Enable **Google Play Android Developer API** in the GCP project linked to the Play developer account.
2. Create a service account with minimum Play Console financial/order permissions to read and acknowledge purchases. Store JSON as a deployment secret.
3. Set:

```text
GOOGLE_PLAY_PACKAGE_NAME=com.yourcompany.everlore
GOOGLE_PLAY_SERVICE_ACCOUNT_JSON={...service account JSON...}
BILLING_ENFORCEMENT_ENABLED=true
```

The API verifies every client purchase token with Google before granting entitlement or Ink. Tokens bind to one Everlore account. **Never** ship the service-account JSON in the Flutter client.

---

## Real-time developer notifications (RTDN)

Play Console → RTDN → Pub/Sub topic → authenticated push subscription to:

```text
POST https://<api-host>/billing/google/rtdn
```

Configure:

```text
GOOGLE_PLAY_RTDN_AUDIENCE=https://<api-host>/billing/google/rtdn
GOOGLE_PLAY_RTDN_SERVICE_ACCOUNT_EMAIL=<push-service-account-email>
```

RTDN updates a purchase only after its first verified client claim has linked the token to an Everlore account (prevents unauthenticated ownership guessing).

---

## HTTP API (summary)

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `GET` | `/billing/catalog` | JWT | Catalog + products + flags |
| `GET` | `/billing/me` | JWT | **Authoritative wallet** (`tier`, `balance`, `profile`, `entitlement`) |
| `POST` | `/billing/google/verify` | JWT | Claim a Play purchase token |
| `POST` | `/billing/simulate-purchase` | JWT | QA checkout (non-production only) |
| `POST` | `/billing/google/rtdn` | OIDC push | Play renewal/cancel/void updates |
| `POST` | `/admin/users/:userId/ink-grants` | Basic admin | Support/QA/promotional grants |

Full shapes: [API.md](./API.md) · [SCHEMAS.md](./SCHEMAS.md).

### Wallet truth

`GET /auth/me` may still serialize a legacy `token_balance` field on the user document. **Do not use that for Membership UI.** Always refresh Ink from `GET /billing/me` (`balance` = sum of `ink_ledger.delta`).

---

## Reserve / settle / release lifecycle

```text
API (WS turn / autofill / image)
  → billingService.reserve(playerId, action, requestId)
       enforcement off → no-op
       enforcement on  → check balance, insert negative ledger row (idempotent key)
  → work runs (generation job, etc.)
  → worker completed / visible-stream fail → settle(reservationId)
  → worker final fail before visible stream → release(reservationId)  # refund
```

Settle records a zero-delta settle marker; release inserts a compensating positive delta. Both are idempotent.

---

## Admin Ink grants

Independent of Play and of the enforcement switch:

```http
POST /admin/users/:userId/ink-grants
Authorization: Basic <admin>
Content-Type: application/json

{
  "amount": 250000,
  "idempotency_key": "qa-creator-initial-grant-2026-07",
  "note": "Founder QA account"
}
```

Max **1,000,000** Ink per grant. Reusing the same idempotency key for the same player does not mint twice. Membership screen reflects the new balance on refresh.

---

## Local / internal simulated checkout

```text
NODE_ENV=development
BILLING_ENFORCEMENT_ENABLED=true
BILLING_SIMULATION_ENABLED=true
```

Membership UI labels checkout as **Test checkout — no charge**. Selecting a plan or pack creates a simulated entitlement or ledger grant; turns deduct immediately.

---

## Pre-launch test checklist

Use an internal testing track and license-test accounts:

- [ ] First subscription grants monthly Ink exactly once
- [ ] Renewal / cancellation / refund RTDN updates active tier
- [ ] Consumable granted once, then consumed/acknowledged
- [ ] Failed pre-token generation **releases** reservation
- [ ] Successful or visible-stream generation **settles** once
- [ ] Refund/void reverses consumable Ink once
- [ ] `simulate-purchase` returns error in production even if flag set

Keep enforcement disabled in non-production unless matching test products and package name are configured there.

---

## Mongo collections

| Collection | Role |
|------------|------|
| `ink_ledger` | Append-only deltas (welcome, reserve, release, settle, purchase, grant, …) |
| `billing_entitlements` | Active subscription tier rows |
| `store_purchases` | Verified Play tokens / acknowledgements |

Indexes: see [DATA_MODEL.md](./DATA_MODEL.md) and `mongo-indexes.ts`.
