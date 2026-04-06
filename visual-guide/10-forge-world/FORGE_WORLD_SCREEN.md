# 10 — Forge World Screen

> **Route:** `/my-worlds/forge` (new) or `/my-worlds/:templateId/forge` (edit)  
> **File:** `lib/features/creator/presentation/forge_world_screen.dart`  
> **Gate:** `lib/features/creator/presentation/forge_world_route.dart` (auth + tier check)  
> **Auth:** Required, Premium/Creator tier

---

## Purpose

A 5-step wizard for creating or editing a world template. Each step has a themed name:
1. **The Essence** — Name, description, world type, content flags
2. **Oracle's Voice** — System prompt / AI personality
3. **Ancient Lore** — World lore, scene tags
4. **Vital Forces** — Stats system, realm flags
5. **Arcane Engine** — Model settings, memory depth, final summary

---

## Layout (Fixed Structure)

```
┌──────────────────────────────────────────────┐
│  ←  ✨ FORGE WORLD               1 / 5     │  ← Header row
├──────────────────────────────────────────────┤
│  THE ESSENCE                                │  ← Step name (18px, w700, parchment)
│  ████████░░░░                               │  ← Progress bar (5 segments)
├──────────────────────────────────────────────┤
│                                            │
│  (Step content — scrollable)                │  ← AnimatedSwitcher (280ms fade)
│                                            │
├──────────────────────────────────────────────┤
│  ⚠ Error message bar              [✕]      │  ← Conditional
├──────────────────────────────────────────────┤
│  [Back]        [Continue →] / [FORGE]       │  ← Bottom nav bar
└──────────────────────────────────────────────┘
```

---

## Header

```
← (back arrow)  ✨ (auto_fix_high icon)  "FORGE WORLD" or "EDIT WORLD"
                                                   (12px, w800, gold, letterSpacing 2.5)
Spacer
"1 / 5" (13px, ash)
```

## Progress Bar

- 5 horizontal segments with 4px gaps
- Active segment: gold, 3px height, gold glow shadow (blur 6)
- Completed segments: goldDim, 2px
- Incomplete segments: void4, 2px
- 300ms animation on step change

## Bottom Navigation Bar

```
Background: void0
Border top: goldDim @15%
Padding: 20px horizontal, 12px top, 16px bottom

Step 0:  [Continue →] only (full width)
Step 1-3: [Back] + [Continue →]
Step 4:   [Back] + [✨ FORGE THIS WORLD]
```

### Back Button
- Height: 48px, padding 20px horizontal
- Background: void3, goldDim @20% border, 12px radius
- Text: "Back", ash, 14px, w600

### Continue Button
- Height: 48px, 12px radius
- Disabled: void3 background, ash text
- Enabled: goldGlow → gold gradient, void1 text, gold glow shadow (blur 14)
- Arrow icon + "Continue"

### Forge Button (Step 4 only)
- Height: 52px, 12px radius
- Background: goldGlow → gold → goldDim gradient (3-stop)
- Shadow: gold @45%, blur 20px, spread 1
- Shows CircularProgressIndicator when submitting
- Icon: auto_fix_high + "FORGE THIS WORLD" (14px, w800, letterSpacing 1.5)

---

## Step 0 — The Essence

```
┌──────────────────────────────────────┐
│  ◉ Define what this world is…      │  ← StepIntro box
├──────────────────────────────────────┤
│  World Name *                       │  ← FormLabel
│  ┌──────────────────────────────┐   │
│  │ e.g. The Shattered Kingdoms  │   │  ← TextField, parchment text, 15px
│  └──────────────────────────────┘   │
│                                     │
│  World Description *                │  ← FormLabel
│  ┌──────────────────────────────┐   │
│  │ Describe the world…          │   │  ← TextField, 4 lines, 14px
│  │                              │   │
│  │                              │   │
│  └──────────────────────────────┘   │
│                                     │
│  WORLD TYPE                         │  ← 11px, gold, w700, letterSpacing 2
│                                     │
│  ┌─── ToggleCard ─────────────────┐ │
│  │ ◉ Conscious Soul          [◉] │ │  ← Active: violet accent
│  │   The world has an AI persona  │ │
│  └────────────────────────────────┘ │
│  ┌─── ToggleCard ─────────────────┐ │
│  │ ◉ Game Master             [ ] │ │  ← Active: cyan accent
│  │   AI acts as neutral narrator │ │
│  └────────────────────────────────┘ │
│                                     │
│  CONTENT                            │  ← 11px, gold, w700, letterSpacing 2
│                                     │
│  ┌─── ToggleCard ─────────────────┐ │
│  │ ◉ Mature Realm            [ ] │ │  ← Active: crimson accent
│  │   Enables mature themes…      │ │
│  └────────────────────────────────┘ │
└──────────────────────────────────────┘
```

### FormLabel Component
- Label text: 13px, w600, parchment
- Required indicator: gold asterisk (*)
- Hint text: 11px, ash

### ToggleCard Component
- Icon circle: 40×40
- Active: activeColor @12% bg circle, activeColor icon
- Inactive: void3 bg circle, ash icon
- Title: 14px, w600 (parchment active, ash inactive)
- Subtitle: 12px, ash
- Toggle switch: 44×26, 13px radius
  - Off: void4 track, parchment thumb, left aligned
  - On: activeColor track, parchment thumb + glow, right aligned

---

## Step 1 — Oracle's Voice

```
┌──────────────────────────────────────┐
│  ◉ The Oracle's Voice is…          │  ← StepIntro box
├──────────────────────────────────────┤
│  ┌─── Tips Box ────────────────────┐ │  ← violet @8% bg, violetDim @30% border
│  │  💡 Tips for a powerful voice    │ │  ← violetBright
│  │  · Describe the narrator's tone  │ │  ← Tip items
│  │  · State the genre and setting   │ │
│  │  · List what to do/not do        │ │
│  │  · Include special rules         │ │
│  └──────────────────────────────────┘ │
│                                     │
│  Oracle's Voice *                   │  ← FormLabel
│  ┌──────────────────────────────┐   │
│  │ System prompt text area       │   │  ← TextField, 10-14 lines, 13px
│  │ (10,000 char max)             │   │     height 1.6, parchment
│  │                              │   │
│  │                              │   │
│  └──────────────────────────────┘   │
│              150 / 10000              │  ← _CharCount (ash or ember if < min)
└──────────────────────────────────────┘
```

### Tips Box
- Background: violet @8%
- Border: violetDim @30%, 10px radius
- Icon: lightbulb_outline, violetBright, 14px
- Title: "Tips for a powerful voice", violetBright, 12px, w600
- Tips: bullet list, 12px, ash

---

## Step 2 — Ancient Lore

```
┌──────────────────────────────────────┐
│  ◉ The Ancient Lore is…            │  ← StepIntro box
├──────────────────────────────────────┤
│  Ancient Lore *                     │  ← FormLabel
│  ┌──────────────────────────────┐   │
│  │ World history, lore, etc.    │   │  ← TextField, 10-14 lines
│  │ (50,000 char max)             │   │
│  └──────────────────────────────┘   │
│              250 / 50000              │  ← _CharCount
│                                     │
│  STORY THREADS                      │  ← Section header
│  Tags for scene types…              │  ← 12px, ash
│                                     │
│  [combat ×] [magic ×] [horror ×]   │  ← _TagChip: gold @10% bg
│                                     │
│  ┌─────────────────────┐ ┌──┐      │
│  │ Add a thread…       │ │ + │      │  ← Input + add button
│  └─────────────────────┘ └──┘      │     add btn: goldDim @15%, gold icon
│                                     │
│  +combat +dialogue +exploration     │  ← Suggested tags: void3 bg, add icon
│  +mystery +romance +horror          │     Filters out already-added
│  +politics +magic +survival         │
└──────────────────────────────────────┘
```

### TagChip
- Background: gold @10%
- Border: goldDim @40%, 6px radius
- Label: gold, 12px
- Remove icon: close, 12px, goldDim

### Suggested Tags
- Background: void3
- Border: void4 @80%
- "+ " add icon: 11px, ash
- Tag text: 12px, ash
- Pre-defined: combat, dialogue, exploration, mystery, romance, horror, politics, magic, survival, stealth

---

## Step 3 — Vital Forces

```
┌──────────────────────────────────────┐
│  ◉ Vital Forces are…                │  ← StepIntro box
├──────────────────────────────────────┤
│  ⚠ At least one Vital Force is      │  ← Warning box: ember border @25%
│  required…                           │     ember icon, ash text
├──────────────────────────────────────┤
│  QUICK ADD                           │  ← Section header
│  [♥ Health ✓] [⚡ Mana] [🧠 Sanity] │  ← Preset pills
│  [🛡 Honour] [💰 Gold] [💪 Strength] │     Added: verdant @10%, check icon
├──────────────────────────────────────┤
│  2 VITAL FORCES                      │  ← Counter
│                                      │
│  ┌─ StatCard ─────────────────────┐  │
│  │ ● health                       │  │  ← Gold dot, parchment name, w600
│  │   0 – 100 · default: 100  ✎ 🗑│  │  ← ash range, edit/delete icons
│  └────────────────────────────────┘  │
│  ┌─ StatCard ─────────────────────┐  │
│  │ ● mana                         │  │
│  │   0 – 100 · default: 100  ✎ 🗑│  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌──────────────────────────────┐    │
│  │  + Add Custom Force           │    │  ← Dashed border, gold text
│  └──────────────────────────────┘    │
├──────────────────────────────────────┤
│  REALM FLAGS                         │  ← Section header
│  Boolean, numeric, or text flags     │  ← 12px, ash
│                                      │
│  ┌─ FlagRow ────────────────────┐   │
│  │  dragon_slayer          ✎ 🗑 │   │  ← parchment name, ash type info
│  │  bool · default: false        │   │     violet @25% border
│  └───────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │  + Add Realm Flag            │   │  ← violet accent
│  └──────────────────────────────┘   │
└──────────────────────────────────────┘
```

### Preset Buttons
- Health (100), Mana (100), Sanity (100), Honour (50, 0-100), Gold (0, 0-999), Strength (10, 1-20)
- Not yet added: void3 bg, goldDim @20% border, icon + name in ash
- Already added: verdant @10% bg, verdant @40% border, check icon + name in verdant

### Stat Editor Bottom Sheet
- Handle: 40×4px, void4, centered
- Title: "Edit/New Vital Force", 17px, w700, parchment
- Fields: Name (required), Min, Max, Default (3-column row), Description
- All inputs: parchment text, _fieldDecoration styling
- Save button: gold ElevatedButton

### Flag Editor Bottom Sheet
- Fields: Key name, Type dropdown (Boolean/Integer/String), Default value, Description
- Save button: gold ElevatedButton

---

## Step 4 — Arcane Engine

```
┌──────────────────────────────────────┐
│  ◉ Fine-tune the arcane engine…     │  ← StepIntro box
├──────────────────────────────────────┤
│  Memory Depth                        │  ← FormLabel
│  How many Echoes the AI considers…   │
│  ──○────────────────────────── 25    │  ← Slider: violet accent, 5-50 range
│  5 (shallow)           50 (deep)     │  ← Slider labels
│                                      │
│  Lore Recall                         │  ← FormLabel
│  How many lore passages per response │
│  ──────○───────────────────── 10     │  ← Slider: cyan accent, 3-20 range
│  3 (focused)         20 (expansive)  │
│                                      │
│  ┌─ Advanced Settings Toggle ──────┐│
│  │  ⚙ Advanced Model Settings  ▼  ││  ← void2 bg, goldDim @20% border
│  └─────────────────────────────────┘│
│                                      │
│  (If expanded:)                      │
│  ┌────────────────────────────────┐ │
│  │  Warning text…                 │ │
│  │  Logic Model         [____]    │ │
│  │  Narration (SFW)     [____]    │ │
│  │  Narration (Mature)  [____]    │ │  ← Only if nsfwCapable
│  │  Summary Model       [____]    │ │
│  └────────────────────────────────┘ │
├──────────────────────────────────────┤
│  ┌─ Forge Summary Card ────────────┐│
│  │  ✨ READY TO FORGE              ││  ← gold icon + label
│  │  World:        My World Name    ││  ← _SummaryRow items
│  │  Type:         Conscious Soul   ││
│  │  Content:      Standard         ││
│  │  Stats:        3 defined        ││
│  │  Scene Threads: combat, magic   ││
│  │  Memory Depth: 25 Echoes        ││
│  │  Lore Recall:  10 passages      ││
│  │  ──────────────────────────── ││
│  │  Your world will be saved as    ││  ← 12px, ash
│  │  a draft…                       ││
│  └─────────────────────────────────┘│
└──────────────────────────────────────┘
```

### Slider Control
- Track height: 3px
- Active track: accent color
- Inactive track: void4
- Thumb: accent color
- Overlay: accent color @15%
- Label shown on drag

### Forge Summary Card
- Background: cardDecoration (gradient #1A1A38 → #0F0F28)
- Border: goldDim @35%
- Shadow: gold @6%, blur 20px
- Header: auto_fix_high icon + "READY TO FORGE" (gold, 11px, w700)
- Rows: label (110px wide, ash) + value (parchment, w500)
- Footer note: 12px, ash
- Divider: void4

---

## Validation

- **canProceed** is checked per step to enable the Continue button
- Step 0: Title and description not empty
- Step 1: Seed prompt ≥ 10 chars
- Step 2: Global lore ≥ 10 chars
- Step 3: At least 1 stat defined
- Step 4: Always valid (uses defaults)

---

## State

- `ForgeWorldCubit` → `ForgeWorldState`
- Fields: `step`, `title`, `description`, `isSentient`, `isNsfwCapable`, `seedPrompt`, `globalLore`, `sceneTags`, `stats`, `flags`, `maxContextMemories`, `maxLoreResults`, `modelLogic`, `modelNarrationSfw`, `modelNarrationNsfw`, `modelSummary`, `isSubmitting`, `canProceed`, `error`, `result`
- On success (`result != null`): navigates to `/my-worlds`
