# Admin ops: the promo becomes a switch, the console shows who is playing  ·  Oct 21 2026

## Why

The app is live on Play (1.0.8, Oct 21 2026). The console can't answer the questions that matter after launch:
- When was a player last here?
- How many people signed up, came back, and installed?

The promo (unlimited Ink with a hidden 500-turn daily cap) is an env flag on the box. Ending it today means SSH and a restart. The Ink allowances the console edits (monthly Ink, daily caps, costs) are dormant while the promo is on, and the console doesn't say so.

## Decisions (orchestrator, Oct 21 2026, from the owner's request)

- The promo moves into the admin-edited `billing_config` document. The env vars stay as the **seed only**: if the document has no `promo` key, `INK_PROMO_UNLIMITED` / `INK_PROMO_DAILY_TURN_CAP` apply, so prod behaves the same until an admin saves. Switching it takes effect within the existing 5 s config cache, with no deploy and no app release (the app already reads `unlimited_ink` from `/billing/me`).
- Turning the promo **off** returns everyone to the tier allowances already shown in the console. Nothing is granted or clawed back: balances are a ledger, and the welcome Ink is granted lazily on the next wallet read as today.
- "Last active" is recorded server-side on authenticated requests, throttled to one write per user per 5 minutes. Before this field existed, last activity is estimated from the newest story event, and the console labels which source it used.
- "Downloads" means Play **installs** from Play's bulk reports (`gs://pubsite_prod_<developer id>/stats/installs/`). This needs the Play permission "View app information and download bulk reports" for the service account the server uses. Until that is granted, the API returns `installs: null` with a reason, and the console says what to do.

## Contract

### `GET /admin/billing/config` and `PUT /admin/billing/config`, adding `promo`

Response (GET and PUT) adds:
```json
{
  "promo": {
    "unlimited": true,
    "daily_turn_cap": 500,
    "source": "config",
    "updated_at": "2026-10-21T09:00:00.000Z",
    "updated_by": "admin"
  }
}
```
- `source` is `"config"` when the document has a `promo` key, otherwise `"env"`. `updated_at` and `updated_by` are `null` for `"env"`.

PUT body adds an optional `promo: { unlimited?: boolean, daily_turn_cap?: integer 1..100000 }`. Only the supplied keys change, as with tiers. Existing fields are unchanged.

### `GET /billing/me`

Unchanged shape. `unlimited_ink` now follows the effective promo (config, else env).

### `GET /admin/metrics?days=30`

`days` is 7–180, default 30. All days are UTC calendar days, oldest first, with every day present (zero-filled).
```json
{
  "generated_at": "ISO",
  "days": 30,
  "totals": { "users": 0, "players_ever": 0, "saves": 0, "story_turns": 0 },
  "active": { "dau": 0, "wau": 0, "mau": 0 },
  "series": [
    { "date": "2026-10-21", "signups": 0, "active_players": 0, "story_turns": 0, "new_saves": 0, "installs": null }
  ],
  "installs": {
    "reason_kind": "forbidden",
    "available": false,
    "reason": "Play hasn't given the server's account access to install reports yet.",
    "last_updated": null,
    "total_user_installs": null
  }
}
```
- `active_players` is distinct players with at least one story turn that day, derived from stored story events, so history is available from day one.
- `dau`/`wau`/`mau` are distinct players with a story turn in the last 1/7/30 days, ending now.
- `installs` per day is Play's "Daily User Installs", or `null` when unavailable. A successful read is cached in memory for 6 hours; a failed read for 15 minutes, then retried on the next request. A fetch failure never fails the endpoint. `installs.reason_kind` is `forbidden` (Play returned 401/403), `no_reports` (no overview CSV yet), `not_configured` (no service account or developer id), `error` (anything else) or `null` on success; `reason` is a fixed sentence per kind and never carries upstream text (added 2026-10-21, server `9385a25`).

### Users, `GET /admin/users` and `GET /admin/users/:id`

Each user record adds:
- `last_active_at`: ISO or `null`;
- `last_active_source`: `"request"` | `"story"` | `null`;
- `story_turns`: integer, for the list.

`GET /admin/users` accepts `sort=last_active|created` (default `created`, newest first). `last_active` sorts nulls last.

### New users and onboarding (added Oct 21 2026, owner: "let us view the new users according to their onboarding info")

Every user record in `GET /admin/users` and `GET /admin/users/:id` adds:
```json
{
  "onboarding": {
    "name": "Mira",
    "gender": "female",
    "interests": ["noir", "epic_fantasy", "romcom"],
    "sign_in": ["phone"],
    "mature_enabled": false,
    "completed": true
  }
}
```
- `name`, `gender` and `interests` come from `preferences.player_name`, `preferences.gender` and `preferences.interests`. Each is `null` or `[]` when the player skipped that step.
- `sign_in` is `providers`.
- `mature_enabled` is `preferences.nsfw_enabled`.
- `completed` is true when a player name is set (step 1 is the only required step).
- Interest values are the stored keys. The console maps them to labels.

`GET /admin/users` also accepts `since=<ISO date>` (users created at or after it), so "new users this week" is one call. It combines with `search` and `sort`.

`GET /admin/metrics` adds, over the same `days` window (users created in it):
```json
{
  "onboarding": {
    "new_users": 0,
    "completed": 0,
    "with_interests": 0,
    "interests": [{ "key": "noir", "count": 0 }],
    "gender": [{ "key": "female", "count": 0 }, { "key": "skipped", "count": 0 }],
    "sign_in": [{ "key": "phone", "count": 0 }, { "key": "google", "count": 0 }]
  }
}
```
Lists are sorted by count, descending. These are what the console shows while installs are unavailable.

## Tasks

| id | lane | paths |
|---|---|---|
| `admin-ops-server` | Sol (`backend`; billing, auth middleware and admin) | `everlore-server/**` |
| `admin-console-redesign` | Claude subagent, design skill (agy's Gemini quota is out until about Oct 24, and this is a design-led rewrite) | `everlore-admin/**` |
| `web-premium` | Claude subagent, design skill, assets via the Codex app (§5) | `everlore-web/**` |
