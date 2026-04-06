# 04 — Home Screen (Your Realms)

> **Route:** `/` | **File:** `lib/features/home/presentation/home_screen.dart` | **Auth:** Required

---

## Purpose

The authenticated user's home screen showing all active world instances ("realms"). This is the primary dashboard after login.

---

## Layout

```
┌──────────────────────────────────────────┐
│  ╭── SliverAppBar (110px expanded) ─────╮ │
│  │                                       │ │
│  │  ◉ EVERLORE    [🧭] [✨] [👤]        │ │  ← Logo mark 34×34, title 16px w800
│  │                                       │ │  ← Header buttons: 36×36 each
│  │  Your Realms                          │ │  ← 24px, w700, parchment
│  ╰───────────────────────────────────────╯ │
│                                            │
│  3 ACTIVE                       Refresh    │  ← sectionHeader + gold link
│                                            │
│  ┌─ WorldCard ──────────────────────────┐  │
│  │  ╭──╮                                │  │  ← Gradient bg (hash-based)
│  │  │◉ │ Title of World     [⋯]         │  │  ← 42px icon, type label
│  │  ╰──╯ Sentient World                 │  │
│  │                                       │  │
│  │  Description text, max 2 lines...     │  │  ← 13px, ash
│  │  ─────────────────────────────────   │  │  ← accentColor @10% divider
│  │  💬 42  🔖 12  [combat]      2h ago  │  │  ← Stats row
│  └───────────────────────────────────────┘  │
│                                            │
│  ┌─ WorldCard ──────────────────────────┐  │
│  │  ...                                  │  │
│  └───────────────────────────────────────┘  │
│                                            │
```

---

## Header Bar

| Element | Size | Action |
|---|---|---|
| Logo circle | 34×34 | — |
| "EVERLORE" text | 16px, w800, letterSpacing 4 | — |
| Explore button (🧭) | 36×36 | → `/templates` |
| My Worlds button (✨) | 36×36 | → `/my-worlds` |
| Profile button (👤) | 36×36 | → `/auth` |

Header buttons: `void2` background, `goldDim @20%` border, `ash` icon, 8px radius.

---

## World Card (`WorldCard` widget)

### Structure

```
┌────────────────────────────────────────┐
│  Container (gradient bg, 18px radius)  │
│  └─ Material (transparent)             │
│     └─ InkWell (18px radius splash)    │
│        └─ Padding (20px all)           │
│           ├─ Row: [Icon] [Title/Type]  │
│           ├─ Description (2 lines)     │
│           ├─ Divider (1px, accent@10%) │
│           └─ Stats Row                 │
└────────────────────────────────────────┘
```

### Visual Properties

| Property | Value |
|---|---|
| Background | LinearGradient from `_realmGradients[hash]` |
| Border | 1px, accentColor @25% |
| Box Shadow | accentColor @6%, blur 24px, offset (0, 4) |
| Border Radius | 18px |
| Splash | accentColor @8% |
| Padding | 20px |
| Margin | 20px horizontal, 8px top, 4px bottom |

### Icon Circle (World Type)

| Property | Sentient | Game Master |
|---|---|---|
| Icon | `psychology_alt` | `auto_stories` |
| Size | 42×42 | 42×42 |
| Background | accent @12% | accent @12% |
| Border | accent @30% | accent @30% |

### Stats Row Components

| Component | Icon | Color |
|---|---|---|
| Events count | `chat_bubble_outline` (13px) | accent @60% |
| Echoes count | `bookmark_outline` (13px) | accent @60% |
| Scene tag | pill badge | accent @10% bg, accent @25% border, 10px text |
| Time ago | text only | ash @60%, 11px |

### Archive Dialog

- Title: "Seal This Realm?" (18px, parchment)
- Content: "This realm will be sealed away…" (14px, ash)
- Actions: "Keep Open" (ash), "Seal Realm" (crimson)

---

## Empty States

### Unauthenticated View

```
╭──╮
│🔒│  ← 80px circle, void2, goldDim @30% border
╰──╯

Your realms await                    ← 22px, w700, parchment
Sign in to access your adventures…   ← 14px, ash

[     Sign In     ]                  ← ElevatedButton → /auth
  Browse Worlds First                ← TextButton → /templates
```

### Empty View (No Instances)

```
╭──╮
│🧭│  ← 90px circle, violet @15% → void2 radial gradient
╰──╯  goldDim @30% border

No realms yet                        ← 22px, w700, parchment
Your first adventure is waiting…     ← 14px, ash

[ 🧭 Explore Worlds ]               ← ElevatedButton.icon → /templates
```

### Loading View

```
◎ (gold progress indicator, 32px)
Summoning your realms…               ← 14px, ash
```

---

## State

- `HomeCubit` → `HomeState`
- State fields: `instances`, `isLoading`, `error`
- Actions: `loadInstances()`, `archiveInstance(id)`
