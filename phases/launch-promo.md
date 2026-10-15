# Launch: the chat app without interactive worlds, with unlimited Ink for the promo  ·  Oct 15 2026

## Scope

Ship the chat-world app from `main` — no interactive worlds ("Walks"). Both repos work on
`release/launch-promo`, cut from `main` (server `6c95e6f`, app `ab0373c`). The interactive work stays on
`feat/interactive-world-markers`. The app's uncommitted W12 changes are parked on `wip/w12-parked`.

Carried over from the feature branch, chat-relevant parts only:
- `4e6c7f8`: every chat model goes through OpenRouter, and embeddings stay on OpenAI.
- The deletion hardening from `73486cc`: delete the save first, drain queued jobs, purge twice, one shared `purgeInstancePayload`.
- Player-facing copy: errors written in the world's voice, and no "AI" shown to the player.

## Promo decisions (owner, Oct 15 2026)

- **No monthly limit and no Ink charge** on any action (story turns, autofill, image previews) while the promo is on.
- **One hidden daily cap: 500 story turns per player per UTC day**, for every tier. This is derived from a ~$6/player/day ceiling. The measured cost is ~$0.006 per turn, and up to ~$0.012 in long stories.
- Free players still keep at most 3 saves (unchanged).
- The promo is switched **on the server** (env). Ending it needs no app release.

## Contract

### Server env

| Var | Default | Meaning |
|---|---|---|
| `INK_PROMO_UNLIMITED` | `false` | When `true`: `reserve()` charges nothing for any action and never returns 402. |
| `INK_PROMO_DAILY_TURN_CAP` | `500` | While the promo is on, the story-turn cap for every tier. It replaces the tier's `daily_story_safety_cap`. |

The promo cap applies whenever `INK_PROMO_UNLIMITED=true`, **whether or not billing enforcement is on**.
Turns are counted by reservation *attempts* that day (UTC), exactly as today: a refunded or failed turn still counts.

### `GET /billing/me` (additive)

```json
{ "...existing fields": "...", "unlimited_ink": true }
```
`unlimited_ink` is always present (`false` when the promo is off). Nothing else changes. The daily cap is **never** sent.

### Daily cap reached — WebSocket error (send / continue / world_action)

```json
{ "type": "error", "code": "STORY_RESTING", "message": "Your story rests until tomorrow.", "resets_at": "2026-10-16T00:00:00.000Z" }
```
`resets_at` is the next UTC midnight, as ISO 8601. The client keys on `code`, never on `message`.
The same code also replaces today's non-promo "Daily story safety limit reached (N turns)…" text. That text must not reach a player.

### Client

- **Header chip** (`EverloreInkBalanceChip`): when `unlimited_ink` is true, show the ink mark followed by the `Icons.all_inclusive` glyph (∞) instead of the number. Semantics label: "Unlimited Story Ink". Tap still opens Membership.
- **Membership** (`BillingScreen`): when `unlimited_ink`, the balance block reads **"Unlimited Ink"** with the subline **"Write as much as you like — for a limited time."** Purchase options are hidden while it's on.
- **Rest screen** on `STORY_RESTING`: a full-screen sheet styled like `_InkReserveDialog` (same art), with this copy:
  - Title: **The Ink Rests**
  - Body: **Your story has run long today, and the quill needs a moment. Everything you've written is kept exactly where you left it.**
  - Line: **Your tale continues at {local time of resets_at, e.g. 5:30 AM}**
  - Button: **Back to my worlds**. It dismisses the sheet and returns to the previous screen.
  - It must not mention limits, turns, Ink cost or "AI". It must not open the paywall.
