# 08 — Template Detail Screen

> **Route:** `/templates/:templateId` | **File:** `lib/features/templates/presentation/template_detail_screen.dart` | **Auth:** None

---

## Purpose

Detailed view of a single world template. Shows expanded description, stat previews, scene type tags, and a prominent "ENTER THIS WORLD" CTA. Creates a game instance and navigate to play.

---

## Layout

```
┌──────────────────────────────────────────┐
│  ← Explore Worlds                   │  ← Pinned SliverAppBar
├──────────────────────────────────────────┘

╭─ Hero Header (180px expanded) ────────────────╮
│ ╭────────────╮                        │  ← Gradient bg (accentColor → void0)
│ │ Ambient ○  │                        │
│ │──────────────────                    │
│ │  ╭──╮  Title          [18+]  ╮   │  ← 64px circle icon, 22px title
│ │  │◉ │              │      │   ← Type badge pill + 18+ badge
│ │  ╰──╯  Sentient World       │   │
│ ╰──────────────────────────────────╯ │
╰────────────────────────────────────────┘

    (scrollable content)

  ┌──────────────────────────────────┐
  │  Description text...               │  ← 15px, ash, height 1.7
  │                                  │
  │  ─── STARTING STATS ───────────── │  ← sectionHeader + gold line
  │                                  │
  │  Stat Name          45 / 100    │  ← Stat bar preview
  │  ████████████████░░░░░░░░░░░░   │  ← verdant/ember/crimson bar
  │  Description text...            │  ← 11px, ash
  │                                  │
  │  ─── SCENE TYPES ────────────── │  ← sectionHeader + gold line
  │                                  │
  │  [combat] [dialogue] [magic]   │  ← Tags: void3, goldDim @20% border
  └──────────────────────────────────┘

  ┌──────────────────────────────────┐  ← Fade gradient overlay
  │                              │
  │  [ 🧭 ENTER THIS WORLD       ] │  ← Pinned bottom CTA
  │                              │     gold bg, void0 text
  └──────────────────────────────────┘   ← 18px vertical, 14px radius
```

### Hero Header

- `SliverAppBar` with `expandedHeight: 180`
- Background gradient: `accentColor @12% → void0`
- Ambient circle: 200×200, accentColor @10%
- World icon circle: 64×64, accentColor @12% bg, @40% border
- Title: 22px, w800, parchment
- Type badge: pill, accentColor @10% bg, @30% border
- 18+ badge: crimson @10% bg, @30% border (conditional)

### Stat Preview Bars

```
Label          45 / 100
████████████████░░░░░░░░░░░░░   ← Gradient bar (verdant/ember/crimson)
Description of stat             ← 11px, ash

Height: 5px track, radius 3px
Color logic:
  ≥ 60% → verdant
  ≥ 30% → ember
  < 30% → crimson
```

### Section Headers

- Gold text label + goldDim divider line
- Style: 11px, w700, gold, letterSpacing 2

### Bottom CTA

- Fixed at bottom via `Positioned` widget
- Fade-out gradient overlay (void1 transparent → void1)
- Button: "ENTER THIS WORLD" with explore icon
- Full width, 18px vertical, 14px radius
- gold background, void0 foreground
- Shows CircularProgressIndicator when creating instance

---

## Navigation

- "ENTER THIS WORLD" → creates instance → `/play/:instanceId`
- Back → `/templates` or `/`

---

## State

- Local `StatefulWidget` state
- Variables: `_template`, `_isLoading`, `_isCreating`
- Fetches template via `TemplateRepository.getById()`
- Creates instance via `HomeRepository.createInstance()`
