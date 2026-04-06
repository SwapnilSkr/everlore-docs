# 05 — Play Screen

> **Route:** `/play/:instanceId` | **File:** `lib/features/play/presentation/play_screen.dart` | **Auth:** Required

---

## Purpose

The core game screen. A real-time AI-driven RPG chat interface where the player interacts with the world through a WebSocket connection. Displays narrative bubbles, world state, player input, and connection status.

---

## Layout

```
┌──────────────────────────────────────────┐
│  ← Title          Realm Active ● [📖] │  ← _PlayHeader (void0)
│    (green/red dot)                         │
├──────────────────────────────────────────┤
│  REALM STATUS  ● ● ●    [▼]        │  ← WorldStateBar (collapsible)
│  Health   ████████████░  75              │
├──────────────────────────────────────────┤
│  ⚠ Error: Something went wrong      ✕ │  ← _ErrorBar (crimson)
├──────────────────────────────────────────┤
│                                       │
│  ┌─ Narrator Bubble ─────────────────┐ │
│  │ ◉ NARRATOR                       │ │  ← Left-aligned, void2 bg
│  │ The shadows twist as you enter│ │
│  │ the ancient tower...              │ │
│  │ [COMBAT] [DIALOGUE]              │ │  ← Scene tag badges
│  └───────────────────────────────────┘ │
│                                       │
│       ┌─ Player Bubble ──────────┐    │  ← Right-aligned, violet gradient
│       │ I draw my sword and     │    │
│       │ attack the shadow!       │    │
│       └──────────────────────────┘    │
│                                       │
│  ◉ The world is weaving your tale…  │  ← Generating indicator (pulsing)
│                                       │
├──────────────────────────────────────────┤
│  ┌──────────────────────────────────┐ │  ← PlayerInput bar (void0)
│  │  What do you do?                ◉ │ │  ← TextField, void2, 18px radius
│  └──────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

---

## Sub-Components

### _PlayHeader

| Element | Style |
|---|---|
| Back button | `Icons.arrow_back_ios_new`, 18px, ash |
| Title | 16px, w600, parchment, ellipsis overflow |
| Connection dot | 6×6 AnimatedContainer; verdant (connected) or crimson (disconnected) |
| Connection label | 11px; "Realm Active" (ash @70%) or "Reconnecting…" (crimson @80%) |
| Lore Tome button | 36×36, void2, gold icon |

### WorldStateBar

| State | Display |
|---|---|
| Collapsed | "REALM STATUS" label + 4 colored mini dots (6×6 each) + down arrow |
| Expanded | Full stat bars: label (90px width) + gradient bar + value |

**Stat Bar Colors:** verdant (≥60%), ember (≥30%), crimson (<30%)

### NarrativeBubble

**Player Bubble:**
- Alignment: centerRight
- Margin: 64px left, 16px right, 8px vertical
- Padding: 16px horizontal, 12px vertical
- Background: violetDim → violet gradient
- Border: violetBright @20%
- Border Radius: topLeft 16, topRight 4, bottomLeft 16, bottomRight 16
- Text: 15px, parchment, height 1.5

**Narrator Bubble:**
- Alignment: left (margin-left: 16px, margin-right: 64px)
- Padding: 16px
- Background: void2
- Border: goldDim @18%
- Border Radius: topLeft 4, topRight 16, bottomLeft 16, bottomRight 16
- Header: "◉ NARRATOR" (gold @50%, 9px, w700)
- Text: MarkdownBody with custom styling (see DESIGN.md markdown section)
- Scene Tag: Pill badge with contextual color

**Generating Indicator:**
- 1200ms pulsing animation
- "The world is weaving your tale…" (italic, pulsing opacity)

### PlayerInput

| Element | Style |
|---|---|
| Container | void0 background, white10 top border |
| Text field | void2, 18px radius; goldDim border (20% unfocused → 60% focused) |
| Hint text | "What do you do?" / "The story unfolds…" / "Reconnecting…" |
| Send button | 44×44 circle; void3 (disabled), gold (active, with glow), or void3 (no text) |
| Generating | Shows CircularProgressIndicator (18px) instead of send icon |

---

## Auto-Scroll Behavior

When new events arrive, the list auto-scrolls to bottom:
- 120ms delay
- 400ms animation
- easeOutCubic curve

---

## State

- `PlayCubit` → `PlayState`
- Key fields: `instanceId`, `instance`, `template`, `events`, `isConnected`, `isLoading`, `isGenerating`, `error`
- WebSocket connection for real-time messaging
- Events list with optimistic UI (player input shown immediately)

---

## Navigation

- Back button → `context.pop()`
- Lore Tome button → `/chronicle/:instanceId`
