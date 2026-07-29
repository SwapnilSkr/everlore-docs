# Billing Feature (Membership & Ink)

Player-facing Story Ink wallet, subscription tiers, and Google Play purchases. Server is the source of truth for balance and entitlements; the client never invents prices or Ink totals.

**Route:** `/membership` → `BillingScreen`  
**Code:** `lib/features/billing/`  
**Server:** [BILLING.md](../../server/BILLING.md) · [API.md — Billing](../../server/API.md)

---

## File structure

```
features/billing/
├── data/
│   └── billing_repository.dart   # /billing/* + in_app_purchase
└── presentation/
    └── billing_screen.dart       # Membership & Ink UI
```

Singleton: `BillingRepository.instance`.

---

## BillingScreen — “Membership & Ink”

| Section | Content |
|---------|---------|
| **Ink balance** | Authoritative `balance` + tier label; note that standard turns cost 1 Ink and failed generations do not consume it |
| **Memberships** | `everlore_premium` (3,000 Ink/mo), `everlore_creator` (6,000 Ink/mo) |
| **Add Ink** | Consumable packs: `everlore_ink_100`, `everlore_ink_350`, `everlore_ink_900` |
| **Test checkout** | Shown when `simulation_enabled` — calls simulate purchase (no charge) |
| **Purchases disabled copy** | When Play catalog is not connected |

Prices displayed on cards come from **Google Play** `ProductDetails` when available; otherwise simulation shows “Test”.

Entry point today: profile / auth UI → `context.push('/membership')`.

---

## BillingRepository APIs

| Client method | HTTP | Notes |
|---------------|------|-------|
| `wallet()` | `GET /billing/me` + `GET /billing/catalog` | Merges wallet with `purchases_enabled` / `simulation_enabled` |
| `products()` | Play `queryProductDetails` | Product IDs listed above |
| `buy(product)` | Play Billing → then `POST /billing/google/verify` | Verifies **before** `completePurchase` |
| `simulatePurchase(productId)` | `POST /billing/simulate-purchase` | Non-production / QA when server flag on |

### Wallet model (`BillingWallet`)

| Field | Source |
|-------|--------|
| `tier` | `/billing/me` |
| `balance` | Ledger-backed Ink total (**not** `user.token_balance`) |
| `monthlyInk` / `dailyStorySafetyCap` | `profile` on wallet response |
| `purchasesEnabled` / `simulationEnabled` | `/billing/catalog` |

### Purchase safety

1. Listen to Play `purchaseStream`.
2. On `purchased` / `restored`, POST verification with `product_id`, `purchase_token`, `kind` (`subscription` \| `consumable`).
3. Only after server grants entitlement, call `completePurchase`.
4. Transient verify failure leaves the purchase pending so Play can redeliver.

---

## Relation to play

Story turns reserve Ink on the **server** WebSocket path before enqueue; the worker **settles** or **releases** the reservation. The Membership screen is for viewing balance and buying more Ink — it does not implement reserve/settle itself. See [BILLING.md](../../server/BILLING.md) and [02-one-turn-journey.md](../../system-guide/02-one-turn-journey.md).

---

## What the client deliberately does NOT do

- Treat `GET /auth/me` → `token_balance` as Story Ink
- Hard-code regional prices (Play owns price tables)
- Acknowledge a purchase before server verification
