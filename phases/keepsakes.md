# Phase: Keepsakes — every picture, film and face the player has made

*Owner decisions, Oct 16 2026. Builds on `phases/media.md` (pictures, films, portraits, passing faces).*

## 1. What it is

A bottom-bar tab named **Keepsakes** replaces the **Worlds** tab (My Worlds, the creator's forge). It holds everything the player has made with "Picture this", "See them say it", "Film this" and the portrait taps, across all their saves. It should feel like a **reel or a post feed**, drawn in Everlore's own design language, not a file browser.

**My Worlds moves into the profile menu** (the top-right profile button) as **"Your worlds"**. It opens the same forge screen as today, with the same tier gate and deep links (`/my-worlds`, `/my-worlds/:id/forge` keep working).

Bottom bar after this change: Explore · Realms · ＋ · **Keepsakes** · Personas.

> ⚠️ `feat/interactive-world-markers` (unmerged) puts **Walks** in the same slot. That merge decides the final bar; nothing here should assume either way.

## 2. What appears

| Shelf | Contains | Never contains |
|---|---|---|
| **Pictures** | `picture` media, `ready` | queued/failed/unseen rows |
| **Films** | `video` media, `ready`: line films and dialogue-free cuts | the scene stills used to make them |
| **Faces** | `portrait` media, `ready`: card portraits and the player's own faces | **passing faces** (`passing_face`), which are never canon (media §3B) |

- Only the player's own media, only `ready`.
- **Voice clips are not keepsakes**; they live on their lines.
- A piece whose save was deleted is gone with it (the media purge already deletes it). A persona face has no save and survives.
- **Superseded portraits:** only a card's **current** portrait is a Face. A repainted-over face doesn't appear.

## 3. The screen

Use `EverloreTheme` tokens throughout:
- the warm-obsidian shell with the champagne-brass accent;
- General Sans for UI text;
- **story excerpts in `EverloreTheme.aiText`** (EB Garamond), never `serifDisplay`.

No vendor names, no "AI", no internal words.

### 3.1 The feed (default)

A vertical feed of **posts**, newest first. Each post:
- **Header row:** the world's cover as a small round crest, the world's title, and under it the save's lead name (or the world kind for a character world) with a relative time ("2h ago"). Tapping the header opens that save.
- **The piece**, full width at its real 9:16 aspect, capped so a post never fills more than ~80% of the screen height. A film shows its poster frame, a muted autoplay loop when at least 60% of it is on screen (one at a time), and a small duration chip.
- **Caption:** the story excerpt the piece came from, 2–3 lines, in EB Garamond, with "more" to expand. For a line film, the caption is the spoken line itself, in quotes, with the speaker's name above it in small caps.
- **Action row:** **Return to this moment** (opens the save scrolled to that turn) and a ⋯ menu. The menu holds **Remove from Keepsakes** (hides it here; the piece stays under its turn in the chat), behind a confirm.

### 3.2 The grid

A toggle at the top right switches between the feed and a 3-column grid of 9:16 tiles, like a profile grid. Films carry a small play glyph and their duration. Tapping a tile opens the viewer at that piece.

### 3.3 Filters

A segmented row under the title: **All · Pictures · Films · Faces**. **All** is pictures and films together. Faces always shows as the grid; there's no feed for faces. A face tile is labelled with the character's name. Tapping it opens the viewer, whose action is **Open their card** instead of Return to this moment.

### 3.4 The viewer (the reel)

Full screen, vertical swipe between pieces in the current filter's order.
- Films play with sound on open and loop. A tap pauses.
- Pictures hold still with a slow, subtle drift (Ken Burns, ≤ 4% scale), which turns off under reduced motion.
- The caption and header overlay the bottom on a dark gradient scrim. A swipe down closes it.
- **Return to this moment** stays one tap away.
- The next two pieces are precached. The viewer never shows a blank frame.

### 3.5 Every edge state

Every state below is designed, not left to fall out of the layout. Nothing shows a blank frame, a black box, a bare spinner, a raw error, or a stretched or awkwardly sparse layout. Each state uses the same shell, the same type and the same art direction as the full feed, so a new player's first look is as polished as a heavy player's.

**The empty states share one composition:**
- a soft illustrated vignette in the house painted style, from the illustrated backgrounds the Realms and Worlds empty states already use (no new art needed);
- one title line in Cinzel display caps, and one or two lines of body copy in the world's voice;
- one primary action, centred in the lower third, within thumb reach.

The filter row stays visible above the state whenever the player has any keepsake at all.

| State | What the player sees |
|---|---|
| **Nothing kept, has saves** | Title *"Nothing kept yet"*. Body: *"Long-press any moment in a story to picture it or film it. What you make gathers here."* Action: **Open your realms** (→ Realms). |
| **Nothing kept, no saves** | The same title. Body: *"Begin a story, then picture or film the moments worth keeping."* Action: **Find a world** (→ Explore). |
| **This filter is empty, others aren't** | A compact version (vignette ~40% of the height), never the full-page state. Pictures: *"No pictures yet. Long-press a moment and choose Picture this."* Films: *"No films yet. Long-press a moment and choose Film this, or long-press a spoken line."* Faces: *"No faces yet. They appear when you picture or film the people in your stories."* Action: **Show all** (→ the All filter). |
| **One to three keepsakes** | The feed shows them normally, then closes with the **end ornament** (below) and a soft nudge: *"Picture another moment"* → Realms. The grid keeps fixed 9:16 tiles, left-aligned; tiles never stretch to fill the width. With fewer than 4 pieces, the grid toggle stays available but the feed is the default. |
| **The end of the list** | A small brass flourish divider and *"That's everything you've kept."* Never a dead stop on blank space. |
| **First load** | Three skeleton posts in the feed's exact shape (header, a 9:16 block, two caption lines) with the house shimmer. The grid shows skeleton tiles. No spinner. |
| **Loading the next page** | One skeleton post at the foot. The list never jumps. |
| **Refreshing** | Pull to refresh, with the brass refresh indicator, on every filter. |
| **First load failed** | The shell's existing calm error composition (as on Realms): *"Your keepsakes are out of reach. Check your connection and try again."* **Try again** reloads in place. |
| **Next page failed** | Pieces already shown stay. The foot shows *"Couldn't reach the rest."* with **Try again**. |
| **A piece's file won't load** | The frame shows a muted parchment tile with a faint crest and *"This keepsake has faded"*, at the same 9:16 size, never black or empty. Its ⋯ menu still offers Remove. The viewer shows the same tile and lets the player swipe on. |
| **A film won't play** | The poster stays, with a small *"Tap to try again"* over it. Without a poster, the frame shows a dark scrim gradient with the play glyph, never black. |
| **Data saver or low power** | Films don't autoplay in the feed. The poster shows with a play glyph, and a tap plays. |
| **Missing details** | The world was removed: the header shows *"A story no longer here"* with a neutral crest, and **Return to this moment** is hidden. No cover: the crest is a brass monogram of the world title's first letter. No excerpt: the caption is omitted, with no empty gap. No save label: that line is omitted. No duration: no chip. |
| **Long text** | World title on one line with an ellipsis. Excerpt clamped to 3 lines with *more*. Speaker name on one line. Face names on two lines max, centred. |
| **The moment no longer exists** | Return to this moment opens the save at the nearest surviving turn, with a calm notice: *"That moment was rewritten. Here's where the story stands."* |
| **Removing the last piece in a view** | The post collapses smoothly and the matching empty state fades in, with no flash of a blank list. |
| **Large text and small phones** | Everything wraps or clamps under the system text scale up to 1.3× without overflow. Action rows wrap rather than shrink their tap targets below 44 pt. |
| **Tablet and wide screens** | The feed is centred at a max width of ~560 pt. The grid uses 4 columns from 700 pt and 5 from 1000 pt. The viewer letterboxes 9:16 on a blurred backdrop of the same piece. |
| **Reduced motion** | No Ken Burns drift, no autoplay, and fades instead of slides. |

Copy in this table is final, and no state names a mechanism ("server", "error code", "media", "AI").

## 4. API

### `GET /media/keepsakes`
Auth required. Lists only `player_id = me`.

Query:
- `kind`: `all` | `picture` | `video` | `portrait` (default `all`). **`all` means pictures and films**: faces aren't moments, so they appear only under `portrait`;
- `before`: the opaque cursor from the previous page's `next_before`, sent back as given;
- `limit`: 1–40, default 20.

Returns:

```ts
{
  items: Array<{
    media_id: string
    kind: 'picture' | 'video' | 'portrait'
    url: string
    poster_url?: string          // video: a frame; absent if none is stored
    duration_ms?: number         // video
    width?: number; height?: number
    created_at: string
    instance_id?: string         // absent for a persona face
    event_id?: string            // absent for a face painted outside a turn
    world: { title: string; cover_url?: string } | null
    save_label?: string          // the lead/persona name, else omitted
    excerpt?: string             // ≤ 280 chars of the turn's prose (picture/cut) or the spoken line (line film)
    speaker_name?: string        // line film only, from the line's attribution
    character?: { id: string; name: string }   // face only
  }>
  next_before?: string           // opaque; absent on the last page. A page is never empty while next_before is present.
}
```

- Sorted by `created_at` descending.
- Excludes:
  - every status but `ready`;
  - `passing_face` and `voice`;
  - superseded portraits;
  - rows the player removed;
  - rows whose save is gone.
- No prompts, model names, costs, asset keys, plans or rejection reasons reach the client.
- `url` and `poster_url` are the same public CDN URLs the chat already uses.

### `POST /media/keepsakes/:mediaId/hide`
Auth required, own media only (otherwise 404). Sets `hidden_from_keepsakes: true` on the row. The piece stays in the chat. Idempotent: 204.

**Index:** `media_assets { player_id: 1, status: 1, created_at: -1 }`.

## 5. Tasks

- **Server** (`keepsakes-server`): the two routes, the index, the poster frame for films. A poster is extracted at assembly when the cut is built, and older films without one return no `poster_url`. Plus an audit:
  - only the owner sees their pieces;
  - exclusions hold;
  - nothing internal is in the payload;
  - paging works;
  - hide works and 404s on another player's piece.
- **App** (`keepsakes-app`):
  - the tab, feed, grid, filters, viewer, and empty and loading states;
  - "Your worlds" in the profile menu;
  - the bar slot change and route moves;
  - tests;
  - a live check on the emulator by the orchestrator.

## 6. Open (not now)
- Saving to the device and sharing outside the app (watermark, privacy, the store's rules for user-generated content).
- Voice clips as keepsakes.
- Liking or pinning a piece.
