# Explore: genres, a mixed For You, and the tour

Agreed with the owner Oct 22 2026. Branch `feat/explore-genres` off `release/launch-promo`, which carries the Play rating prompt (`089c637`), so both ship in the next release.

## Why

- **There is no genre.** A world has a narration voice (`narrative_style`), and voices roll up into five interest families (`STYLE_FAMILY`). That is not the shelf the owner thinks in: Horror sits inside Dark, and the seven anime-voiced Epic worlds count as Cozy.
- **Worlds and Characters filter the wrong set.** `discover_screen.dart` loads 20 worlds a page and filters `isCharacter` on the phone, so the Characters tab shows only the characters among what happens to be loaded. Genre filters built the same way would inherit the bug.
- **For You is not a mixture.** `listPublished` sorts every interest match first, then by recency, so a horror fan sees a wall of horror and everyone sees the newest batch first.

## Decisions

1. **`genre` is authored data**, separate from voice: `romance`, `epic`, `dark`, `horror`, `modern`, `cozy`. A world is data; the server never infers it from a title or a word list.
2. **The tabs are genres** (owner, revised Oct 22): For You · Romance · Epic · Dark · Horror · Modern · Cozy. Worlds and Characters stop being tabs.
3. **For You stays one mixed feed** (owner): interest-weighted but interleaved across genres.
4. **One filter icon** beside search opens a sheet for everything else: kind (worlds or characters), interests (narration voices) and sort. Nothing else sits on the page, so it does not clutter.
5. **The server filters.** Genre, kind and interests are query parameters; the phone never filters a page it was handed.

## Contract

### World template

| Field | Type | Notes |
|---|---|---|
| `genre` | `'romance' \| 'epic' \| 'dark' \| 'horror' \| 'modern' \| 'cozy'` | Present on every listed world after the backfill. Returned in the public projection. |

- `POST /templates` and `PUT /templates/:id` accept an optional `genre`, validated against the six keys. When it is missing, the server derives it from `narrative_style`: the `horror` voice gives `horror`, any other voice gives its family (`romance_charged` → `romance`, `epic_adventure` → `epic`, `atmospheric_dark` → `dark`, `modern_everyday` → `modern`, `cozy_playful` → `cozy`), and no voice gives `epic`. One shared helper does this, used by create, update and the backfill.
- `genre` is **not** a meaningful-edit field (`MEANINGFUL_WORLD_FIELDS`): moving a world between shelves does not send it back to review.
- The catalog files gain `"genre"`, set from their batch folder, and `seed-catalog.ts` writes it.

### `GET /templates`

New optional parameters, combined with the existing `page`, `limit` and `search`:

| Param | Values | Effect |
|---|---|---|
| `kind` | `world` \| `character` | Filters on `kind` in the database query. |
| `genre` | one of the six keys | Filters on `genre`. An unknown value is a 400. |
| `styles` | comma-separated voice keys | Filters on `narrative_style` ∈ list (the sheet's interests). Unknown keys are a 400. |
| `sort` | `foryou` (default) \| `new` | `new` is plain recency, ignoring interests. |

- **No `kind` and no `genre` (For You):** an interleaved order. Each genre forms its own queue, sorted by interest score (exact voice, then family) and then recency. The feed takes from the queues in a weighted round-robin: a genre the player picked gets 2 places per round, any other genre 1, and the starting genre rotates daily so the top of the feed changes. No genre appears more than twice in a row while others remain. The order is computed over the listed set (a few hundred worlds today) and paged from it. This needs a rethink past a few thousand worlds; note it in the code.
- **With `kind`, `genre` or `styles`:** interest score, then recency, as now (unless `sort=new`).
- `total` reflects the filters.

### `GET /templates/genres`

```json
{ "genres": [ { "key": "romance", "label": "Romance", "worlds": 3, "characters": 30,
                "styles": { "dark_romance": { "worlds": 0, "characters": 10 }, "flirty": { "worlds": 2, "characters": 9 } } }, ... ] }
```

Counts of listed worlds per genre, kind and voice, after the same discovery and moderation filter as the list. The app hides a genre tab with no worlds, and the sheet offers only the voices present in the current genre (all genres on For You) and, when Show is set to Worlds or Characters, only voices with that kind (revised after the emulator run, Oct 22). It is public, cached for 60 s, and ordered by the six-key order above.

**The genre list is server-driven.** The keys and labels live in one server constant, and the app builds its tabs from this response. A new genre (for example Sci-fi) is a server change plus its worlds; it needs no app release. The app falls back to For You only if the call fails.

### Backfill

`scripts/backfill-genre.ts`, a dry run by default and `--apply` to write:
- Catalog worlds (slug present in `launch/catalog/<batch>/`): genre = batch (`romance`, `epic`, `dark`, `horror`, `modern`).
- Everything else: the derivation helper above.
- It prints a count per genre and never touches a world that already has a genre.
- Run on dev, then on prod by the orchestrator (AWS rule).

## App

- **Tab bar:** For You, then the genres from `GET /templates/genres` in the order it returns them. It scrolls sideways if needed, uses the existing tab style and shows only genres with worlds. One-word labels: Romance, Epic, Dark, Horror, Modern, Cozy.
- **Filter icon**, beside the search icon in the header. It opens a bottom sheet (root navigator, like every sheet) with three groups:
  - **Show:** All · Worlds · Characters (single choice).
  - **Interests:** the narration voices present in the current tab, as multi-select chips using the interests-picker labels (Tsundere, K-drama, Grimdark and so on).
  - **Sort:** For you · Newest.
  - The sheet has Clear and Apply buttons. While any filter is set, the icon shows a small brass dot and a one-line "2 filters · Clear" sits under the tabs. Nothing else is added to the page.
- **Loading:** a tab change or Apply resets to page 1 with `genre`, `kind`, `styles` and `sort` sent to the server. The old cards stay visible while the new page resolves, as search already does. The client-side `isCharacter` filter goes.
- **Filters per tab:** filters are kept per tab for the session. Interests that do not exist in the newly opened tab are dropped quietly.
- **Empty state:** the existing empty card with a "Clear filters" action.
- **Model:** `WorldTemplate.genre` (nullable for old caches; a null genre shows only on For You).

## Tour (coachmarks)

- **Arrival arc, tabs beat** (`GuideIds.discoverTabs`): new copy, now about genres ("For You mixes everything. Each tab is a genre.").
- **New beat in the arrival arc**, anchored to the filter icon: worlds or characters, and the interests you like.
- **New one-beat arc `exploreGenres`:** the same two points as a single beat, fired once on the next Explore visit, for players who already finished arrival. A player new to arrival never sees it (arrival covers it), and it is recorded at start like every arc.
- **Rules:** beats wait for the tab bar and the first cards to have painted, using the existing arrival wait.
- The debug rehearsal (`guide_rehearsal.dart`) replays both.

## Lanes

- **Server** (`Engine: muse`, non-sensitive): the genre field, helper, validation, list parameters, interleave, `/templates/genres`, the backfill, the seed change and the audits. `audit:seed-catalog` and a new genre check must be shown to bite.
- **App** (`agy`): the genre tabs, the filter sheet, loading, the model field, and the tour beats and copy.
- **Orchestrator:** catalog JSONs gain `genre`; review; emulator run (empty genres hidden, filters reach the server, the tour fires once); backfill on prod; release with the rating prompt.

## Open

- The website's world pages keep their own genre labels; syncing them to `genre` is a later, separate change.
