# Private and public worlds, with admin approval · and the open forge  ·  Oct 15 2026

Ships with the launch (`release/launch-promo` on server and app; `feat/world-review` on the admin console).

## Decisions (owner, Oct 15 2026)

- A creator chooses who can enter a published world:
  - **Private** ("Just for you"): playable at once, visible only to its creator. No review.
  - **Public** ("For every traveller"): reviewed by an admin first. It is listed in Discover only once approved. The creator can play it while it waits.
- An admin either **approves** a public world or **asks for changes**, with a note. A world that needs changes stays playable by its creator, is not listed, and is resubmitted by publishing it as public again.
- **A meaningful edit to an approved public world sends it back to review** and out of Discover. Existing saves on it keep working.
- **The open forge:** while the server switch is on, every tier can make worlds and characters (create, autofill, image previews). When it's off, only Premium/Creator can, as today. Private/public review applies either way.
- Worlds already published, and the official **Everlore** catalog seed, count as public and approved.

## Data (on `world_templates`)

| Field | Type | Notes |
|---|---|---|
| `visibility` | `'private' \| 'public'` | Set on publish. **Absent** on drafts. |
| `review_status` | `'pending' \| 'approved' \| 'changes_requested'` | Only when `visibility: 'public'`. `$unset` it when a world becomes private. Never write `null`. |
| `review_note` | string ≤ 500 | Only with `changes_requested`. `$unset` otherwise. |
| `submitted_at`, `reviewed_at` | Date | |
| `reviewed_by` | string | The admin username. |

**Legacy rule:** a published world with no `visibility` field is treated as `public` + `approved`, both in code and in the Discover filter. No migration is needed.

**Listed in Discover** means: `is_published && public && approved && moderation_status !== 'hidden'`, plus the existing block filters.

**Meaningful fields** — a change to any of them on an `approved` public world sets `review_status: 'pending'`:
- `title`, `description`, `image_url`
- `seed_prompt`, `global_lore`, `opening_line`
- `protagonist`, `seed_cast`
- `is_nsfw_capable`, `kind`, `is_sentient`

Changes to other fields (stats, flags, model preferences, limits) don't trigger review. An edit while `pending` stays `pending`. An edit while `changes_requested` stays `changes_requested` until the world is republished.

## API

### `POST /templates/:id/publish` (owner only) — body `{ visibility?: 'private' | 'public' }`

- Missing `visibility` means **private**, the safe default for older app builds.
- **private:** `is_published: true`, `visibility: 'private'`, `$unset` review fields.
- **public:** `is_published: true`, `visibility: 'public'`, `review_status: 'pending'`, `submitted_at: now`, `$unset review_note`.
  - Publishing an already-approved public world as public again, with no meaningful change, keeps it `approved`.
- Calling it again changes visibility. Going public to private delists the world at once.
- **Response:** the template's `{ visibility, review_status?, review_note? }`.

### Reading a world

`GET /templates/:id`, and starting a new save, by anyone but the creator:
- **404** if the world is private, or public but not approved. This is the same pattern as a hidden world.
- Existing saves keep working.

### Owner listing

`GET /templates/mine/list` includes `visibility`, `review_status` and `review_note`.

### The open forge

- Env: `FORGE_OPEN_TO_ALL_TIERS` (default `false`).
- `serializeUser` (`/auth/me` and login responses) gains `can_forge: boolean`. It is `true` when the switch is on, or the effective tier is premium/creator.
- Every server forge check (template create, image generate, image upload, autofill, `mine/quota`) uses the **same** helper.
- The app decides only from `can_forge` and never re-derives it from tier. If `can_forge` is missing (an older server), fall back to the tier check.

### Admin (behind `requireAdmin`, inside the admin route group)

- **`GET /admin/worlds/review`:** public worlds with `review_status: 'pending'`, oldest `submitted_at` first. Each item has:
  - `_id`, `title`, `description`, `image_url`
  - `seed_prompt`, `global_lore`, `opening_line`
  - `kind`, `is_sentient`, `is_nsfw_capable`, `narrative_style`, `protagonist`
  - `creator { _id, username }`
  - `submitted_at`, `was_approved_before: boolean`
- **`POST /admin/worlds/:worldId/approve`:** sets `approved`, `reviewed_at` and `reviewed_by`, and unsets `review_note`.
- **`POST /admin/worlds/:worldId/request-changes`** with `{ note: string, 1–500 }`: sets `changes_requested`, `review_note`, `reviewed_at` and `reviewed_by`.

Both return 409 unless the world is public and pending.

## App copy (final, word for word)

### Publish sheet

- **Title:** Who can enter this world?
- **Subline:** You can change this later.
- **Option 1:** "Just for you" / "Ready to play now. Only you will see it."
- **Option 2:** "For every traveller" / "We read every shared world first, usually within a day. You can play it while you wait."
- **Button:** "Publish"

### Status chips

| State | Chip |
|---|---|
| unpublished | Draft |
| private | Just for you |
| pending | Awaiting the seal |
| approved | Shared with all |
| changes_requested | Needs changes (the note shown under the chip, in quotes) |

### Saving a meaningful edit to an approved public world

- **Confirm title:** "Send it to be read again?"
- **Body:** "Changes to a shared world are read again before it returns to Discover. You can keep playing it meanwhile."
- **Buttons:** "Save and resubmit" / "Keep editing"

### Your Worlds notice, once per review

- **changes_requested:** "{title} needs changes", then the note, then the button "Edit world".
- **approved:** "{title} is now shared with every traveller".

### World menu

"Make it just for you" / "Share with every traveller". The second opens the publish sheet with public preselected.
