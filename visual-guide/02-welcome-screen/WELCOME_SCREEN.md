# 02 — Welcome Screen

> **Route:** `/welcome` | **File:** `lib/screens/welcome_screen.dart` | **Auth:** None

---

## Purpose

The landing page for unauthenticated users. Presents the app with a fantasy-themed invitation to sign up or browse worlds. No technical jargon — pure fantasy language.

---

## Layout

```
┌────────────────────────────────┐
│  (ambient violet glow top-left)│
│                                │
│         ╭────────╮             │
│         │ ◉ Icon │             │  ← Logo circle 90×90, gold border, glow
│         ╰────────╯             │
│                                │
│          EVERLORE              │  ← 34px, w800, gold, letterSpacing 8
│                                │
│  Every choice echoes through   │  ← 14px, ash, letterSpacing 0.5
│         eternity               │
│                                │
│  ──── Spacer ────              │
│                                │
│  ◉ A living world remembers…  │  ← Feature rows (violet, gold, cyan)
│  ◉ Your story, your choices   │
│  ◉ Infinite crafted worlds    │
│                                │
│  ──── Spacer ────              │
│                                │
│  ╔═══════════════════════════╗ │
│  ║  BEGIN YOUR JOURNEY      ║ │  ← ElevatedButton, gold bg, void0 text
│  ╚═══════════════════════════╝ │
│                                │
│  ┌───────────────────────────┐ │
│  │  Explore Worlds           │ │  ← OutlinedButton, white20 border
│  └───────────────────────────┘ │
│                                │
│  (ambient gold glow bottom-R)  │
│                                │
└────────────────────────────────┘
```

---

## Components

### Ambient Background

Two positioned radial gradient circles:
- **Top-left**: violet @ 12% → transparent, 360×360
- **Bottom-right** (15% from bottom): gold @ 8% → transparent, 280×280

### Logo Circle

- Size: 90×90
- Shape: circle
- Gradient: RadialGradient (void3 → void1)
- Border: 1.5px gold @ 50% opacity
- Box shadow: gold @ 15%, blur 40, spread 5
- Icon: `Icons.auto_stories`, gold, size 40

### Title

- Text: "EVERLORE"
- Style: 34px, w800, gold, letterSpacing 8.0

### Tagline

- Text: "Every choice echoes through eternity"
- Style: 14px, ash, letterSpacing 0.5

### Feature Rows (`_FeatureRow`)

Three rows, each with:
- Icon circle: 36×36, shape circle, color @ 12% background
- Icon: 18px, themed color
- Text: 14px, ash, height 1.4

| # | Icon | Color | Text |
|---|---|---|---|
| 1 | `psychology_alt` | violet | "A living world that remembers everything" |
| 2 | `history_edu` | gold | "Your story, written by your choices" |
| 3 | `explore` | cyanBright | "Infinite worlds crafted by our community" |

### CTA Buttons

**Primary (ElevatedButton):**
- Label: "BEGIN YOUR JOURNEY"
- Style: 15px, w800, letterSpacing 1.5
- Background: gold, Foreground: void0
- Padding: 18px vertical
- Radius: 14px, no elevation
- Action: `context.push('/auth')`

**Secondary (OutlinedButton):**
- Label: "Explore Worlds"
- Style: 14px, letterSpacing 0.5
- Foreground: ash
- Border: white20
- Padding: 16px vertical
- Radius: 14px
- Action: `context.push('/templates')`

---

## Animation

Single `AnimationController`:
- Duration: 1000ms
- Curve: easeIn
- Wraps entire content in `FadeTransition` (opacity 0 → 1)

---

## Navigation

- "BEGIN YOUR JOURNEY" → `/auth`
- "Explore Worlds" → `/templates`

---

## State

- `SingleTickerProviderStateMixin` for fade animation
- No Cubit/Bloc
