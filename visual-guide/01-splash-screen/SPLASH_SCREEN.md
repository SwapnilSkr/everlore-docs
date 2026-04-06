# 01 — Splash Screen

> **Route:** `/splash` | **File:** `lib/screens/splash_screen.dart` | **Auth:** None

---

## Purpose

The splash screen is the app's entry point. It shows an animated logo and tagline, checks if the user is authenticated via cached credentials, and routes to either Home (authed) or Welcome (unauthed).

---

## Layout

```
┌────────────────────────────┐
│                            │
│         (void0 bg)         │
│                            │
│        ╭──────────╮        │
│        │  Glow orb │        │  ← Pulsing radial glow (gold + violet)
│        │ ╭────────╮│        │
│        │ │ ◉ Icon ││        │  ← auto_stories icon, gold, 36px
│        │ ╰────────╯│        │  ← Outer ring, goldDim 50% pulse
│        ╰──────────╯        │  ← Inner circle, RadialGradient void3→void1
│                            │
│        EVERLORE            │  ← gold, 30px, w800, letterSpacing 8.0
│                            │
│    A Living World Awaits   │  ← ash, 13px, letterSpacing 2.0
│                            │
│           ◎                │  ← CircularProgressIndicator, 24px, goldDim 60%
│                            │
└────────────────────────────┘
```

---

## Components

### Logo Stack (centered)

| Layer | Element | Size | Style |
|---|---|---|---|
| 1 (back) | Glow backdrop | 140×140 | Circle, boxShadow: gold@15%·blur80 + violet@10%·blur120, animated pulse |
| 2 | Outer ring | 100×100 | Circle, border: goldDim@50%·pulse, 1px |
| 3 | Inner circle | 80×80 | Circle, RadialGradient(void3→void1), border: gold@60%, 1.5px |
| 4 | Icon | — | `Icons.auto_stories`, gold, 36px |

### Title Text

```
Font: 30px, w800, letterSpacing 8.0
Color: gold
Content: "EVERLORE"
```

### Tagline

```
Font: 13px, letterSpacing 2.0
Color: ash
Content: "A Living World Awaits"
```

### Loading Indicator

- `CircularProgressIndicator`, 24px, strokeWidth 1.5
- Color: `goldDim @ 60%`

---

## Animation Sequence

```
0ms ──────────────── 1080ms ────── 1800ms ── 2400ms ──→
     [Logo Fade+Scale]  [Tagline]   [Wait]   [Route]
     easeIn, 0→60%      easeIn               _checkAuth()
     0.7→1.0 scale      50→100%
     
     ◄─── glow pulse (2200ms, repeat reverse) ───►
          0.4→1.0 opacity
```

| Animation | Interval | Curve | From → To |
|---|---|---|---|
| Logo opacity | 0–60% of 1800ms | easeIn | 0 → 1 |
| Logo scale | 0–70% of 1800ms | easeOutCubic | 0.7 → 1.0 |
| Tagline opacity | 50–100% of 1800ms | easeIn | 0 → 1 |
| Glow pulse | 2200ms repeat | easeInOut | 0.4 → 1.0 |

---

## Navigation Logic

1. `_logoController.forward()` completes (1800ms)
2. Wait additional 600ms
3. Call `AuthService.getCachedUser()`
4. If user found → `context.go('/')` (Home)
5. If no user → `context.go('/welcome')` (Welcome)

---

## State

- Uses `TickerProviderStateMixin` (two animation controllers)
- No Cubit/Bloc — purely local state
- Controllers disposed on widget dispose
