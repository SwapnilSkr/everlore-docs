# 07 — Browse Templates Screen

> **Route:** `/templates` | **File:** `lib/features/templates/presentation/browse_screen.dart` | **Auth:** None

---

## Purpose

A searchable catalog of published world templates. Users can search by keyword and tap to view details.

 Accessible without authentication.

---

## Layout

```
┌──────────────────────────────────────────────┐
│  ←  Explore Worlds                 │  ← Pinned SliverAppBar, void0 bg
│     Choose your next adventure    │
├──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  🔍 Search for a world...          ✕ │  ← Search bar: void2, goldDim border,14px r
└──────────────────────────────────────────────┘

  3 WORLDS FOUND                    ← sectionHeader (11px, gold, letterSpacing 2)

  ┌─ WorldCard ───────────────────────────┐
│  │  ╭──╮                        │  ← Gradient bg per card
│  │  │◉ │ Title                  │  ← 16px, w700, parchment
│  │ ╰──╯ Type (Sentient/GM)       │
│  │          18+ badge (if nsfw)   │
│  │                             │
│  │  Description...                │  ← 13px, ash, maxLines 2
│  │                             │
│  │  [combat] [dialogue] [magic]  │  ← Scene tag pills (void4, white10 border)
│  └──────────────────────────────────┘
│  ┌──────────────────────────────────┐
│  ... more cards ...                  │
└──────────────────────────────────────────────┘
```

### World Card (Browse)

| Element | Style |
|---|---|
| Background | void2, 16px radius, Accent color border @15% |
| Icon circle | 40×40, accentColor @10% bg, @30% border |
| World type icon | `psychology_alt` (Sentient) or `auto_stories` (GM) |
| Title | 16px, w700, parchment |
| Type label | 11px, accent color @80% |
| 18+ badge | crimson @10% bg, @30% border (conditional) |
| Description | 13px, ash, maxLines 2 |
| Scene tags | Chips: void4, white10 border, 10px |
| Chevron | ash @50% |

### Accent Colors

- Sentient worlds → `violetBright`
- Game Master worlds → `cyanBright`

---

## States

### Loading
- CircularProgressIndicator (28px, gold)
- "Discovering worlds…" (italic, ash)

### Error
- wifi_off icon (40px, ash)
- "Could not reach the realm" (16px, w600, parchment)
- Error message (12px, ash)
- "Try Again" button (gold)
### Empty
- explore_off icon (48px, goldDim)
- "No worlds found" (16px, w600, parchment)
- "Try a different search term." (13px, ash)

---

## State

- Local `StatefulWidget` state
- No Cubit/Bloc
- Variables: `_templates`, `_isLoading`, `_error`, `_searchController`
