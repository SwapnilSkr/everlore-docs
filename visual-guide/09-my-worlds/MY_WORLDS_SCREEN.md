# 09 — My Worlds Screen

> **Route:** `/my-worlds` | **File:** `lib/features/creator/presentation/my_worlds_screen.dart` | **Auth:** Required (Premium/Creator)

---

## Purpose

Creator dashboard showing all worlds the user has created. Drafts and published worlds are displayed in separate sections. Includes a tier gate for free users and an auth gate for unauthenticated users.

---

## Gate Views

### Unauthenticated Gate

```
╭──╮
│🔒│  ← 80px circle, void2, goldDim @30% border
╰──╯
Sign in to forge worlds           ← 20px, w700, parchment
Only authenticated creators…          ← 14px, ash

[   Sign In   ]                       ← gold button → /auth
```

### Upgrade Gate (Free Tier)

```
╭──╮
│✨│  ← 90px circle, gold @18% → void2 radial gradient, goldDim border, gold glow
╰──╯
ASCEND TO FORGE                     ← 20px, w800, gold, letterSpacing 2

World creation is granted to      ← 14px, ash, height 1.6
Premium and Creator tier…

┌─ Upgrade Features Card ─────────┐
│  ◉ Create & Publish Worlds       │  ← Feature list (5 items)
│  ◉ Custom AI Personalities       │
│  ◉ Design Stat Systems           │
│  ◉ Control Narrative Depth       │
│  ◉ Forge Scene Threads           │
└─────────────────────────────────┘
  ℹ Upgrade available through profile    ← goldDim info banner
```

Each upgrade feature:
- 34px circle icon (gold @10% bg)
- Title: 14px, w600, parchment
- Subtitle: 12px, ash
- Dividers: `#1E1E3C` color, 1px

---

## Main Layout (Creator View)

```
┌──────────────────────────────────────────┐
│  ←  ✨ MY WORLDS           [↻]      │  ← Header: void0, 110px expanded
│     Your Creations                        │  ← 22px, w700, parchment
├──────────────────────────────────────────┘

  ┌── DRAFTS ────────── ember color ──┐
  │                               │
  │  ┌─ MyWorldCard ──────────┐ │
  │  │  [amber top stripe]     │ │  ← 3px gradient bar
  │  │  ╭──╮ Title  [DRAFT]   │ │  ← Draft badge (ember)
  │  │  │◉ │ type   [Edit]    │ │
  │  │  ╰──╯  [Release →]    │ │  ← Action buttons
  │  └──────────────────────────┘ │
  │                               │
  └───────────────────────────────┘

  ┌── PUBLISHED ────── verdant color ─┐
  │                               │
  │  ┌─ MyWorldCard ──────────┐ │
  │  │  [verdant top stripe]    │ │  ← 3px gradient bar
  │  │  ╭──╮ Title  [LIVE]     │ │  ← Live badge (verdant)
  │  │  │◉ │ type   [Edit]    │ │
  │  │  ╰──╯  [Preview →]     │ │  ← Action buttons
  │  └──────────────────────────┘ │
  │                               │
  └───────────────────────────────┘

  ┌──────────────────────────────┐
  │  ✚ FORGE WORLD               │  ← FAB: pill shape, gold gradient
  └──────────────────────────────┘   ← 52px height, gold glow shadow
```

### MyWorldCard

| Element | Style |
|---|---|
| Top stripe | 3px, gradient (accent @60% → @20%) |
| Background | LinearGradient (void2 → void1 @95%) |
| Border | ember @30% (draft) or verdant @30% (published) |
| Icon circle | 40×40, accent @12% bg, @30% border |
| Title | 16px, w700, parchment |
| Type badge | pill, accent @10% bg, @30% border, 10px |
| 18+ badge | crimson @10% bg, @30% border, 10px w700 |
| Status badge | ember (draft) or verdant (live) pill |
| Description | 13px, ash, height 1.45, maxLines 2 |
| Info chips | void4 @50% bg, 10px text |
| Action buttons | icon 14px + text 13px, w600 |

### Forge FAB

- Height: 52px
- Padding: 20px horizontal
- Background: goldGlow → gold gradient
- Shadow: gold @40%, blur 18px, spread 1
- Icon: add, void1
- Text: "FORGE WORLD", void1, 12px, w800, letterSpacing 1.5

### Publish Dialog

- Title: "Release This World?" with public icon (gold)
- Content: Quote + description
- Actions: "Keep Hidden" (ash), "Release to the Realm" (gold)

---

## State

- `MyWorldsCubit` → `MyWorldsState`
- Fields: `worlds`, `drafts`, `published`, `isLoading`, `publishingIds`, `error`
- `FutureBuilder<User?>` for auth check
- Actions: `load()`, `publish(id)`, `clearError()`
