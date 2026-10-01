# W3 — The fight: design brief

*Phase 1, workstream 3. Written Oct 1 2026 by the orchestrator. Server contract W3-C is shipped (`everlore-server` `b675999`, `audit:duel-contract`). The UI agent builds to this; deviations go in its report, not in the code.*

## What is wrong today (from the code and the owner)

`world_duel.dart` (`DuelStage`, 461 lines) advances on a **1.7s timer** (`_play`, ~L92). It draws two `StageFigure`s with name plates, the question as an inscription, the prose as a `StageFloat`/`StageBubble`, and two `StageMeter`s, then ends on an `_OutcomePanel` (`StagePanel`). The owner: *"it looks like two cards moving on their own."* Nothing invites the player in, nothing *hits*, and the bars are plain.

## The contract the client now receives (W3-C)

On the duel: `player_side` (`challenger|defender|null`), `near_side`.
On each beat: `striker` (`near|far`), `awaits_player` (bool), `impact` (`blocked|glancing|solid|decisive`), plus the existing `actor`, `action` (prose), `said`, `toll`, `challenger_vigour`, `defender_vigour`, `decisive`.

**Add these to the Dart models** (`domain/interactive_world.dart` `WorldDuel`, `WorldDuelBeat`), tolerant of their absence: an older server means `player_side: null`, `striker` from `actor` vs challenger, `impact` = `solid`, and no `awaits_player`.

## Invariant (non-negotiable)

**The tap only paces the fight.** It releases the next `awaits_player` beat, which is already written and already decided. The client never picks, skips into or alters an outcome, and never decides anything from `toll` beyond animating it. `Skip` jumps to the verdict exactly as today.

## Layout

- **Backdrop:** the arena painting full-bleed (`StageBackdrop`), darkened to 55% with a radial vignette, so the fighters and the shield carry the light.
- **Near fighter** (the `near_side`, i.e. the player's when they fight): **small, bottom-left.** Cut-out height ≈ 34% of screen height, anchored bottom-left with an 8% inset, slightly cropped by the bottom edge.
- **Far fighter: small, top-right.** ≈ 30% of screen height (farther = smaller), anchored top-right below the safe area.
- A fighter with no portrait keeps the existing standard (`StageStandard`), at the same corner rect, never centred behind anything.
- **Vigour bars** sit by their owner: the near bar along the bottom-left above the safe area, the far bar top-right under the name. Each bar shows the name on it.
- **No panel during the fight.** No `StagePanel` and no reading box. The question is shown once as an inscription during *Prepare*, then fades away.
- **Prose is a caption strip:** one line, max 2, centred at ~58% height, `EverloreTheme.aiText` 16/1.35 over a soft dark band (not a box), and it fades in and out with each beat. `said` keeps the bubble rule already in `composeDuel` (the server drops it when the prose quotes it).

## Flow

1. **Prepare** (on open, `player_side != null`): the fighters fade in at their corners; the question inscription shows for 2.2s, then fades. The **Verdict shield** (bundled asset `assets/interactive/ui_verdict_shield.webp`, ~132px) sits centre-screen and:
   - rotates slowly (one turn per 9s);
   - breathes in scale 0.94 ↔ 1.06 on a 1.6s loop, `Curves.easeInOutSine`;
   - throws a glint sweep (a diagonal highlight band) every 3.2s;
   - pulses a soft gold ring outward behind it.

   A one-line caption under it: "Raise your shield" (world voice; no mechanics words).
2. **The player's strike** (`awaits_player == true`): the shield waits. **Tap the shield:**
   - The shield snaps to scale 1.25 and fades (180ms).
   - The near figure **lunges** along the diagonal toward the far fighter. It translates ~55% of the way and scales up ×1.15 over 260ms (`Curves.easeInCubic`), with a slight lean (rotate −4°) and motion-blur smear (a 2–3 frame afterimage trail at decreasing opacity).
   - **Impact** at the far fighter. The size follows `impact`:
     - `blocked`: a spark burst of 8 particles, a 2px screen shake, the far figure flinching back 6px, and a dull "deflect" ring. No damage number.
     - `glancing`: 14 particles, a 4px shake, a 90ms white flash on the far figure, a small damage number (−toll) floating up.
     - `solid`: 22 particles plus a slash streak across the far figure, an 8px shake, a 120ms flash, the figure knocked back 14px and recovering, and a bold damage number.
     - `decisive`: a slash plus a radial burst, a 14px shake, a short hit-stop (90ms freeze), a red-gold flash, the far figure knocked back 24px and dropping to 70% opacity and desaturated, and the largest number.
   - Return: the near figure eases back to its corner (360ms, `easeOutCubic`).
   - The bar drains (see below). The caption shows the beat's prose.
   - Then the **opponent's** beats play automatically, and the shield reappears for the next `awaits_player` beat.
3. **The opponent's strike** (`awaits_player == false`, `striker == far`): **no shield.** After 700ms the far figure lunges at the near fighter (mirror of the above, toward the bottom-left), with the same impact scale applied to the near side. A `blocked` reads as the player's guard holding.
4. **A Verdict the player only watches** (`player_side == null`): no shield at all. Beats auto-play every ~2.2s with the same lunge and impact language. A tap anywhere advances sooner. `Skip` remains.
5. **Verdict:** after the last beat, a 600ms beat of stillness, then the loser's figure fades to 35%, and the verdict card rises. Reuse `_OutcomePanel`/`StagePanel` here; this is the only place a panel appears. Continue → W4's "never left hanging" return.

All particles, slashes, flashes, shakes and numbers are **code-drawn** (`CustomPainter` / implicit animations). No raster effects. Respect `MediaQuery.disableAnimations`: there the lunge becomes a crossfade, with no shake and no particles.

## The vigour bars (replace `StageMeter`'s plain bar for duels)

- **Segmented:** 10 segments with 2px gaps, inside a brass-edged frame (code-drawn: a thin gold stroke and a dark inner track).
- **Two layers:** the live fill drops instantly on impact, and a **ghost** fill (pale gold, 60% alpha) lags behind and drains to match over 550ms after a 250ms delay. This is the trailing-damage read.
- **Hit flash:** the frame flashes white for 100ms on damage.
- **Low:** below 25%, the fill shifts to deep crimson and pulses its glow (1.2s loop).
- **No numbers on the bar** (the damage numbers float off the fighter instead).

## Assets

`sprite_ui_verdict_shield`: generated Oct 1 (Codex), chroma green, matted by the orchestrator and **bundled** into `everlore/assets/interactive/` (not the world manifest). The orchestrator adds it and the `pubspec.yaml` asset entry before this task starts. Nothing else is raster.

## Files

`presentation/world_duel.dart` (rewrite the stage; keep `DuelStage`'s public constructor), new `presentation/stage/stage_vigour_bar.dart`, `presentation/stage/stage_strike_fx.dart` (particles, slash, flash, shake, number), `presentation/stage/stage_shield.dart`, the barrel `stage.dart`, tokens in `stage_tokens.dart` (add a `StageFightLayout` beside `StageDuelLayout`), and the two Dart model classes. **Not** `interactive_world_screen.dart` (W4 owns the return path).

## Done when

- `flutter analyze lib/features/interactive/` is clean and `flutter test` passes.
- The orchestrator drives all three Verdicts on the emulator (one the player fights, one where the lead is Cassian, one watched) and sees: the shield only on the player's beats, a lunge and impact scaled by `impact`, ghost-draining bars, no panel until the verdict, and no jank. Report includes screenshots at Prepare, mid-impact and Verdict.
