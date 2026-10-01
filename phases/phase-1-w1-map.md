# W1 — The map: design brief

*Phase 1, workstream 1. Written Oct 1 2026 by the orchestrator. The UI agent builds to this; deviations go in its report, not in the code.*

## What is wrong today (measured from the code)

- **Every place is the same marker:** a 15px circle with a 1.6px brass border and a radial glow, 30px overall (`world_map_view.dart` `_Landmark._pin`, ~L484). *Here* fills it brass, *sealed* adds an **8px** `Icons.lock_rounded`, and *selected* scales it ×1.18. Nothing else distinguishes a place you can reach now from one three roads away, or a goal from a backwater.
- **Rumoured places are not drawn at all** (`_visible` drops them). The "you've heard of somewhere" beat, the strongest lure a map has, is invisible.
- **The way on** is a chip pinned under the title (`_WayChip`), detached from the place it points to.
- **The place card** (`_SelectionCard`, `interactive_world_screen.dart` ~L1102) is a flat `0xE6100C0A` rounded rectangle (r18) with a 116px **`BoxFit.contain`** thumbnail (letterboxed on a 0x33 black box), a title, a paragraph, and one Material-ish action. No hierarchy, no art direction, no state.
- **Selection is instant:** the camera doesn't move and the card pops in with no transition.

## Principles

1. **State is read by shape and motion first, colour second.** A player should tell the six states apart in greyscale.
2. **Only two things on the map ever move:** *you are here* and *the way on*. Everything else is still. Motion is the lure.
3. **Markers are chrome, drawn in code** (`CustomPainter`), constant size on screen at every zoom, as today. Decided: no raster marker sprites (the Sept 6 finding: raster stickers fail at map scale).
4. **One small raster ornament is allowed: the wax seal.** A wax seal is a silhouette, not a texture, so it holds at 28px. It comes from Codex (`.agents/assets/w1-map-chrome.md`).
5. **Use theme tokens:** `EverloreTheme.gold`, `goldDim`, `parchment`, and the type helpers. A raw hex is allowed only where the existing file already defines that colour; add new tokens to the theme rather than inlining them.

## The six marker states

All markers are painted on a **44×44 logical-px** canvas, centred on the place's point, with the 52px hit target kept. Sizes below are on-screen.

| State | Condition (all derivable client-side from data it already has) | Look | Motion |
|---|---|---|---|
| **Here** | `id == state.currentLocationId` | Full brass medallion, 26px: sweep-gradient ring (gold → goldDim → gold), a 2px inner bevel highlight top-left, a filled dark-bronze centre carrying a small **walking-figure / boot glyph**, and a soft ground shadow ellipse beneath (the medallion "sits" on the map) | A slow **breathing halo**: a ring expanding from 26 → 44px while fading 0.5 → 0, on a 2.4s loop, `Curves.easeOut` |
| **Reachable** | in `state.travelLocationIds` | Brass ring medallion, 22px, open dark centre with a small **diamond** pip | Still |
| **Known road** | in `state.quickTravelLocationIds`, not reachable | 18px ring at 70% opacity, no pip | Still |
| **Way on** | the place `_wayOn` points to, if any | The reachable look plus a **gold chevron/banner pennant** hanging above it (code-drawn), replacing the floating chip | Pennant sways ±4° on a 3s loop; the chip at the top is removed |
| **Sealed** | `visibilityFor(flags) == sealed` | 20px **dark iron** ring (no brass) with the **wax seal** raster (28px, red-brown) pinned over its lower-right quadrant | Still. On the turn it unseals: the seal cracks (scale 1 → 1.15 + fade, 450ms), and the ring warms from iron to brass over 600ms |
| **Rumoured** | `visibilityFor(flags) == rumoured` **and** in `revealedIds` (rumoured is currently filtered out — draw it) | No ring: a hand-inked **question-mark sigil** in faded ink, 16px, plus a dotted "ink-blot" circle | Still. Title shown in italic and dimmed; card copy says what was heard, never the name of a mechanic |

**Labels:** keep the current rule that hides names under the title band. Labels use `EverloreTheme.ui` at 12.5/600 for *here* and *way on*, 12/500 for the rest, and 12 italic at 55% for *rumoured*. Keep the double shadow.

**Selected (any state):** the marker scales ×1.15 (180ms, `easeOutBack`) and gains a thin rotating dashed ring 6px outside it (one revolution per 8s). Only the selected marker gets it.

## Selecting a place

1. Tap: the camera **eases** to centre the place, shifted up so it sits in the top 40% of the viewport above the card. Use `Matrix4Tween` on the `TransformationController`, 420ms, `Curves.easeInOutCubic`, keeping the current zoom (clamped as `_frameOn` already does).
2. The card slides up from 24px below with a fade (280ms, `easeOutCubic`). When the selection changes while the card is open, the content cross-fades (180ms) and the card does not drop and rise again.
3. Tapping empty map dismisses the card (slide down, 200ms).

## The place card

A parchment card replaces the dark box. It is **not** a Material card.

```
┌──────────────────────────────────────────┐  ← 16px side margins, max width 520, bottom 16 + safe area
│ [ scene painting, 16:9, BoxFit.cover,    │  ← full-bleed top, r14 top corners only, 6% vignette
│   bottom 35% gradient to the card ]      │
│   REALM NAME · small caps 11/600 gold    │  ← over the painting's bottom-left, 12px inset
│   Place Title  (serifDisplay 22)         │
├──────────────────────────────────────────┤
│ ◆ state line  e.g. "You stand here"      │  ← 12.5 ui, colour per state, with the state's
│                     "One road away"      │     marker glyph at 14px leading it
│                     "Sealed — <reason>"  │
│                     "Only rumoured"      │
│ Description, aiText 15/1.45, max 3 lines │  ← "more" expands to 6 lines in place, never a dialog
│ People known here: ◯◯◯ (avatars 24px)    │  ← only cast already met, from data the client has; omit row if none
│                                          │
│ [   PRIMARY ACTION — full width, 48h  ]  │  ← brass-filled pill, label in ui 15/700 dark;
└──────────────────────────────────────────┘     busy state keeps the label + a 16px spinner
```

- **Background:** a parchment texture (Codex asset `texture_parchment_card`, tileable, 512px), multiplied over `#1A140F` at 92% so it stays dark-theme. Hairline brass border at 35%, plus a 12px soft shadow.
- **Exactly one primary action.** In order: *Enter* (here and enterable), *Travel* (reachable), *Take the known road* (quick travel), otherwise none. A sealed place shows no button; its state line is the reason. The secondary "known road" travel, when both apply, is a text button under the primary.
- **No thumbnail letterboxing.** If the painting is missing, the header collapses to the title block on parchment, with no empty grey box.
- `textScaler`: the card grows; the description clamps; the action never leaves the screen, and the card scrolls internally above 70% of the viewport height.

## Travel

Tap *Travel*: the card slides down, then a **route line** draws from here to there along the straight segment (a dashed brass line, 2px, drawn over 500ms). The camera follows the line's end (600ms, `easeInOutCubic`), then hands over to the arrival walk (W2, not this task). Today's `_travel` and `travel_transition.dart` stay the entry points; only the map side changes here.

## Out of scope for W1

The arrival walk and room (W2), the fog look (keep it), map plates (keep them), and any server change. **Any data the card needs that the client doesn't already have is a contract question for the orchestrator, not an improvisation.**

## Done when

- `flutter analyze lib/features/interactive/` is clean.
- On the emulator, the orchestrator sees all six states on the Iron Verdict map, at opening zoom and zoomed out, and the select → camera ease → card → travel flow is smooth with no jank. Screenshots of each state go in the report.
