# Everlore — UI & Design Specification

> **Version:** 1.0.0  
> **Platform:** Flutter (iOS, Android, Web, macOS, Windows, Linux)  
> **Theme:** `NexusTheme.dark` — Dark Fantasy RPG  
> **Last Updated:** 2026-04-06

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Navigation & Routing](#2-navigation--routing)
3. [Color Palette](#3-color-palette)
4. [Typography](#4-typography)
5. [Spacing & Layout](#5-spacing--layout)
6. [Component Library](#6-component-library)
7. [Motion & Animation](#7-motion--animation)
8. [Screen Map](#8-screen-map)
9. [State Management](#9-state-management)
10. [Accessibility](#10-accessibility)

---

## 1. Design Philosophy

Everlore uses a **Dark Fantasy RPG** aesthetic. The UI language is themed around fantasy concepts:

- Realms → game instances / worlds
- Echoes → AI-generated memories
- Forge → world creation tool
- Lore Tome → chronicle / history viewer
- Vital Forces → stat system
- Oracle's Voice → system prompt / AI personality
- Arcane Engine → model configuration

**Core Principles:**

| Principle | Implementation |
|---|---|
| **Immersive theming** | Every label, empty state, and interaction uses in-universe language |
| **Deep blacks** | Background hierarchy uses five `void*` shades from near-black to dark-indigo |
| **Gold accent** | `#D4A843` gold is the primary action color; all CTAs, active states, and headers use gold |
| **Ambient glows** | Radial gradients and box-shadows create a "magical aura" effect on key elements |
| **Gradient surfaces** | Cards use subtle `LinearGradient` backgrounds rather than flat fills |
| **Minimal chrome** | No shadows/elevation; borders are thin, gold-tinted, semi-transparent |
| **Fantasy copy** | "Summoning your realms…", "Seal This Realm?", "The world is weaving your tale…" |

---

## 2. Navigation & Routing

The app uses **GoRouter** (`go_router: ^17.0.0`) with named routes. All page transitions use **CupertinoPageTransitionsBuilder** (horizontal slide) on every platform.

### Route Map

| Route | Name | Screen | Auth Required |
|---|---|---|---|
| `/splash` | `splash` | SplashScreen | No |
| `/welcome` | `welcome` | WelcomeScreen | No |
| `/` | `home` | HomeScreen | Yes |
| `/auth` | `auth` | AuthScreen | No |
| `/play/:instanceId` | `play` | PlayScreen | Yes |
| `/chronicle/:instanceId` | `chronicle` | ChronicleScreen | Yes |
| `/templates` | `templates` | BrowseTemplatesScreen | No |
| `/templates/:templateId` | `template_detail` | TemplateDetailScreen | No |
| `/my-worlds` | `my_worlds` | MyWorldsScreen | Yes (Premium+) |
| `/my-worlds/forge` | `forge_world` | ForgeWorldRoute → ForgeWorldScreen | Yes (Premium+) |
| `/my-worlds/:templateId/forge` | `edit_world` | ForgeWorldRoute → ForgeWorldScreen | Yes (Premium+) |

### Navigation Flow

```
[Splash] → auth check → [Welcome] (unauthed) or [Home] (authed)
[Welcome] → [Auth] → [Home]
[Home] ↔ [Templates] → [Template Detail] → [Play]
[Home] ↔ [My Worlds] ↔ [Forge World]
[Play] ↔ [Chronicle]
```

---

## 3. Color Palette

All colors are defined as static constants in `EverloreTheme` (`nexus_theme.dart`).

### Void Scale (Backgrounds)

| Token | Hex | CSS | Usage |
|---|---|---|---|
| `void0` | `#07070E` | `rgb(7, 7, 14)` | Deepest background, app bars, input bars |
| `void1` | `#0D0D1C` | `rgb(13, 13, 28)` | Scaffold background |
| `void2` | `#13132A` | `rgb(19, 19, 42)` | Card backgrounds, input containers |
| `void3` | `#1A1A38` | `rgb(26, 26, 56)` | Raised card backgrounds, button fills |
| `void4` | `#252548` | `rgb(37, 37, 72)` | Input field fills, hover states, inactive tracks |

### Gold Scale (Primary Accent)

| Token | Hex | CSS | Usage |
|---|---|---|---|
| `gold` | `#D4A843` | `rgb(212, 168, 67)` | Primary actions, logo, section headers, active borders |
| `goldDim` | `#8A6820` | `rgb(138, 104, 32)` | Borders, dim gold indicators |
| `goldGlow` | `#F0C86A` | `rgb(240, 200, 106)` | Bright gold for gradients, glow effects |

### Violet Scale (Secondary)

| Token | Hex | CSS | Usage |
|---|---|---|---|
| `violet` | `#7C3AED` | `rgb(124, 58, 237)` | Sentient world accent, secondary actions, phone auth |
| `violetDim` | `#4C1D95` | `rgb(76, 29, 149)` | Player bubble gradients, dim violet |
| `violetBright` | `#A855F7` | `rgb(168, 85, 247)` | Bright violet highlights |

### Cyan Scale (AI / Tertiary)

| Token | Hex | CSS | Usage |
|---|---|---|---|
| `cyan` | `#0891B2` | `rgb(8, 145, 178)` | Game Master world accent, tertiary |
| `cyanBright` | `#22D3EE` | `rgb(34, 211, 238)` | AI text highlights, links |

### Semantic Colors

| Token | Hex | CSS | Usage |
|---|---|---|---|
| `parchment` | `#E8DCC8` | `rgb(232, 220, 200)` | Primary text color |
| `ash` | `#9CA3AF` | `rgb(156, 163, 175)` | Secondary text, hints, captions |
| `ember` | `#D97706` | `rgb(217, 119, 6)` | Warning, mid-range stats |
| `verdant` | `#059669` | `rgb(5, 150, 105)` | Success, high stats, published status |
| `crimson` | `#DC2626` | `rgb(220, 38, 38)` | Danger, low stats, errors, delete |

### Utility Colors

| Token | Hex | Usage |
|---|---|---|
| `white10` | `#1AFFFFFF` (10% white) | Dividers, subtle borders |
| `white20` | `#33FFFFFF` (20% white) | Inactive stars, dim borders |
| `white40` | `#66FFFFFF` (40% white) | Medium-opacity elements |

### Scene Tag Colors

| Tag | Color | Hex |
|---|---|---|
| combat | crimson | `#DC2626` |
| intimate | rose | `#EC4899` |
| exploration | verdant | `#059669` |
| existential | cyanBright | `#22D3EE` |
| cosmic | violetBright | `#A855F7` |
| dialogue | gold | `#D4A843` |
| mundane | ash | `#9CA3AF` |

### Memory Type Colors

| Type | Label | Color |
|---|---|---|
| relationship | Bond | `#EC4899` (rose) |
| promise | Oath | `#D4A843` (gold) |
| lore | Lore | `#22D3EE` (cyanBright) |
| observation | Sight | `#A855F7` (violetBright) |
| emotion | Feeling | `#F97316` (orange) |
| secret | Secret | `#DC2626` (crimson) |

### Realm Card Gradient Palettes

Cards are assigned gradients by `instanceId.hashCode % 5`:

| Index | Colors | Accent |
|---|---|---|
| 0 | `#1A0A2E → #0D1A3E` | violet |
| 1 | `#1A1500 → #0E1A10` | gold |
| 2 | `#0A1A1A → #0A0E2A` | cyanBright |
| 3 | `#1A0D0D → #1A0A22` | `#EC4899` (rose) |
| 4 | `#0A120A → #121A0A` | `#34D399` (emerald) |

---

## 4. Typography

### Predefined Text Styles

| Style Name | Size | Weight | Letter Spacing | Color | Height | Usage |
|---|---|---|---|---|---|---|
| `displayTitle` | 32 | w800 | 3.0 | gold | 1.1 | Logo text, hero titles |
| `screenTitle` | 20 | w700 | 1.5 | parchment | — | App bar titles |
| `sectionHeader` | 11 | w700 | 2.0 | gold | — | Section labels, "STARTING STATS", "SCENE TYPES" |
| `cardTitle` | 17 | w600 | 0.3 | parchment | — | Card titles |
| `body` | 15 | normal | — | parchment | 1.6 | Body text |
| `bodyDim` | 13 | normal | — | ash | 1.5 | Secondary body text |
| `caption` | 11 | normal | 0.5 | ash | — | Captions, timestamps |
| `aiText` | 15 | normal | 0.1 | `#CBEAF5` | 1.7 | AI narration text |

### Logo Style

The EVERLORE logo uses: `fontSize: 30-34, fontWeight: w800, letterSpacing: 8.0, color: gold`

### Theme TextTheme Mapping

| Theme Token | Style |
|---|---|
| `displayLarge` | displayTitle |
| `titleLarge` | cardTitle |
| `bodyLarge` | body |
| `bodyMedium` | bodyDim |
| `bodySmall` | caption |

### Markdown Styling (Narrator Bubbles)

| Element | Style |
|---|---|
| H1 | 22px, w700, gold |
| H2 | 19px, w700, gold |
| H3 | 17px, w600, parchment |
| H4 | 16px, w600, parchment |
| Strong | w700, parchment |
| Italic | italic, aiText |
| Code | monospace, 13px, goldGlow bg void4 |
| Blockquote | italic, ash, left border goldDim 3px |
| Links | cyanBright, underline |

---

## 5. Spacing & Layout

### Border Radii

| Element | Radius |
|---|---|
| Cards / major containers | 16px |
| Elevated buttons | 12px |
| Outlined buttons | 12px |
| Input fields (theme) | 12px |
| Forge input fields | 10px |
| Chip / tag badges | 20px (pill) |
| Tab indicators | 10px |
| Dialogs | 20px |
| Error / info bars | 10px |
| Player input container | 18px |
| FAB | 26px (pill) |

### Padding Patterns

| Context | Padding |
|---|---|
| Screen horizontal | 20-32px |
| Card internal | 14-20px |
| Button vertical | 14-18px |
| Input content | 16px horizontal, 12-14px vertical |
| Header top bar | 24px horizontal, 16px top |
| Section headers | 24px horizontal, 16px top, 8px bottom |
| Sliver list items | 20px horizontal, 8px top |

### Component Sizes

| Component | Width | Height |
|---|---|---|
| Logo circle (splash) | 80-100px | 80-100px |
| Logo circle (welcome) | 90px | 90px |
| Logo mark (header) | 34px | 34px |
| Header icon button | 36px | 36px |
| World type icon circle | 40-42px | 40-42px |
| Avatar circle (auth) | 80px | 80px |
| Tier badge | auto | ~24px |
| Scene tag badge | auto | ~20px |
| Send button | 44px | 44px |
| FAB | auto | 52px |
| Progress bar track | full | 5-6px |

---

## 6. Component Library

### 6.1 Cards (`cardDecoration`)

```
Background: LinearGradient (#16163A → #0F0F28)
Border: 1px goldDim @ 35% opacity
Border Radius: 16px
Color Base: void2
```

### 6.2 Cards with Glow (`cardDecorationGlow`)

```
Background: LinearGradient (#1E1A40 → #120F2A)
Border: 1px gold @ 50% opacity
Box Shadow: gold @ 10% opacity, blur 20px
```

### 6.3 Input Fields (`inputDecoration`)

```
Fill: void4
Border: 1px goldDim @ 30% opacity
Focused Border: 1.5px gold
Radius: 14px
Content Padding: 16px horizontal, 14px vertical
Hint Style: #5A5A80, 14px
Label Style: ash
```

### 6.4 Buttons

**Elevated Button (Primary)**
```
Background: gold
Foreground: void1
Text: w700, letterSpacing 1.0, 14px
Radius: 12px
Padding: 16px vertical, 24px horizontal
```

**Outlined Button (Secondary)**
```
Foreground: gold
Border: 1px goldDim
Radius: 12px
Padding: 14px vertical, 20px horizontal
```

### 6.5 Stat Bar

```
Track: void4, 5px height, radius 3px
Fill: Gradient (color@70% → color@100%), with boxShadow color@35%
Color Logic:
  ≥ 60% → verdant
  ≥ 30% → ember
  < 30% → crimson
```

### 6.6 Toggle Card (Forge)

```
Inactive: void2 background, goldDim @ 15% border
Active: activeColor @ 8% background, activeColor @ 45% border
Toggle switch: 44x26px, radius 13px
  Off: void4 track, parchment thumb
  On: activeColor track, parchment thumb with activeColor glow
```

### 6.7 Tab Buttons (Chronicle)

```
Inactive: void2 background, goldDim @ 15% border
Active: gold @ 12% background, goldDim @ 60% border
Icon: 14px, text: 13px
Radius: 20px (pill)
```

### 6.8 Dialogs

```
Background: void2
Border: 1px goldDim @ 30-40% opacity
Radius: 20px
Title: 17-18px, w700, parchment
Content: 14px, ash, height 1.5
Actions: TextButton with ash (cancel) or crimson/gold (confirm)
```

### 6.9 Error Banner

```
Background: crimson @ 10%
Border: crimson @ 30%
Radius: 10px
Icon: error_outline, 16px, crimson
Text: 13px, crimson
```

### 6.10 Loading Shimmer

```
Gradient: void2 → void4 → void2
Animated sweep, 1500ms repeat
Radius: 4px
```

### 6.11 Forge Progress Bar

```
5 segments, animated
Active: gold, 3px height, gold glow shadow
Completed: goldDim, 2px
Incomplete: void4, 2px
```

---

## 7. Motion & Animation

### Page Transitions

All platforms use **CupertinoPageTransitionsBuilder** (horizontal iOS-style slide).

### Splash Screen Animations

| Animation | Duration | Curve | Details |
|---|---|---|---|
| Logo fade in | 0 → 60% of 1800ms | easeIn | Opacity 0 → 1 |
| Logo scale | 0 → 70% of 1800ms | easeOutCubic | Scale 0.7 → 1.0 |
| Tagline fade | 50% → 100% of 1800ms | easeIn | Opacity 0 → 1 |
| Glow pulse | 2200ms | easeInOut, repeat reverse | Opacity 0.4 → 1.0 |

### Welcome Screen

| Animation | Duration | Details |
|---|---|---|
| Full screen fade | 1000ms | easeIn curve |

### Play Screen

| Animation | Duration | Details |
|---|---|---|
| Auto-scroll to bottom | 400ms | easeOutCubic, 120ms delay |
| Connection indicator | 600ms | AnimatedContainer color change |

### Forge World

| Animation | Duration | Details |
|---|---|---|
| Step transition | 280ms | FadeTransition via AnimatedSwitcher |
| Progress bar | 300ms | AnimatedContainer |
| Toggle cards | 200ms | AnimatedContainer |
| Next button | 200ms | AnimatedContainer gradient/color |

### Generating Indicator (Narrative Bubble)

| Animation | Duration | Details |
|---|---|---|
| Pulse | 1200ms | easeInOut, repeat reverse |
| Icon opacity | +0.4 * pulse | gold @ 40-80% |
| Text opacity | +0.3 * pulse | ash @ 50-80% |

---

## 8. Screen Map

Detailed screen documentation is in the numbered sub-folders:

| # | Screen | Route | File |
|---|---|---|---|
| 01 | Splash | `/splash` | [01-splash-screen/](./01-splash-screen/) |
| 02 | Welcome | `/welcome` | [02-welcome-screen/](./02-welcome-screen/) |
| 03 | Auth | `/auth` | [03-auth-screen/](./03-auth-screen/) |
| 04 | Home | `/` | [04-home-screen/](./04-home-screen/) |
| 05 | Play | `/play/:instanceId` | [05-play-screen/](./05-play-screen/) |
| 06 | Chronicle | `/chronicle/:instanceId` | [06-chronicle-screen/](./06-chronicle-screen/) |
| 07 | Templates Browse | `/templates` | [07-templates-browse/](./07-templates-browse/) |
| 08 | Template Detail | `/templates/:templateId` | [08-template-detail/](./08-template-detail/) |
| 09 | My Worlds | `/my-worlds` | [09-my-worlds/](./09-my-worlds/) |
| 10 | Forge World | `/my-worlds/forge` | [10-forge-world/](./10-forge-world/) |

---

## 9. State Management

The app uses **flutter_bloc** (`flutter_bloc: ^9.1.1`) with Cubits:

| Cubit | Feature | Purpose |
|---|---|---|
| `HomeCubit` | Home | Load/archive world instances |
| `PlayCubit` | Play | WebSocket chat, events, world state |
| `ChronicleCubit` | Chronicle | Events, memories, tab switching, CRUD |
| `MyWorldsCubit` | Creator | Load/publish worlds |
| `ForgeWorldCubit` | Creator | Multi-step form, validation, creation |

### Data Flow

```
UI (Screen) → Cubit.method() → Repository (API/Cache) → State emitted → BlocBuilder rebuilds
```

### Networking

- REST API via `ApiClient` (package:http)
- WebSocket via `web_socket_channel` for real-time play
- Auth tokens stored in `flutter_secure_storage`
- Offline cache via `sqflite`

---

## 10. Accessibility

### Color Contrast

- Primary text (`parchment #E8DCC8`) on `void1 (#0D0D1C)` → ~13:1 contrast ratio
- Secondary text (`ash #9CA3AF`) on `void1` → ~6.5:1 contrast ratio
- Gold (`#D4A843`) on `void1` → ~7:1 contrast ratio

### Touch Targets

- All interactive elements meet minimum 44x44px touch targets
- Header icon buttons: 36x36px with 8px borderRadius tap area
- List items use full-width InkWell with borderRadius

### Semantics

- Tooltip widgets on icon-only buttons
- Icon buttons have semantic tooltips
- Selectable text in narrator bubbles (MarkdownBody selectable: true)

### Focus Indication

- Input fields: gold border on focus (1.5px width)
- Player input: goldDim border transitions from 20% → 60% opacity on focus
