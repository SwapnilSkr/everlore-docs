# a2-room-audit: Arrival and the room panel

Read-only. No file other than this report was edited. Nothing was run on a device; every claim is from the widgets as they are written. The fixer owns the change. Ranked by what the player actually sees.

The owner’s complaint matches the code. The walking view is a 1.2s shiver buried under a near-opaque mist, then deleted in one frame while the mist snaps open. The parchment they like is real; what fills it is several different jobs at the same size.

---

## 1. Arrival — what moves, for how long, what hitches

Two different entries exist. Only one of them plays `TravelTransition`.

### A. Travel (`_travel`, not haste)

`interactive_world_screen.dart:587-622`, overlay at `750-757`.

One `setState` does all of this before any image is ready:

- `_veiling = true` — `WorldMist` mounts at full strength, above the walk.
- `_travellingTo = destination.id` — `TravelTransition` mounts and its controller calls `forward()` in the field initializer (`travel_transition.dart:30-33`). The clock starts on the same frame as the mist, not after decode.
- `_acting = true`.

Stack, bottom to top (`689-758`):

1. The map (the view is still `_View.map`).
2. `TravelTransition` (only while `_travellingTo != null`).
3. `WorldMist` (while `_veiling`).

The mist wins. Its base is `Color(0xE60A0908)` (`world_mist.dart:68`), about 90% opaque, plus a darker gradient and five blurred clouds. The walk shows through as a dim ghost, or not at all.

Then, still under that veil:

1. `awaitWorldFrame` on the destination painting only, no cut-outs, no decode size (`597-600`). Ceiling 8s (`world_frame.dart:14`).
2. `_act(type: 'move')` — the round trip. If anyone is standing there, this also opens a conversation before the room is shown (see defect 2).
3. If the move landed and the place can be entered: `await _awaitSceneFrame()` again (`606-607`, `655-683`), now with cut-outs.
4. One `setState` sets `_travellingTo = null` and `_view = scene` (`609-612`). The walk widget is unmounted with no exit. The scene (or the conversation, if step 2 addressed someone) mounts underneath the mist in that same frame.
5. Another `setState` clears `_acting` (`616-621`).
6. `_liftVeil` (`640-642`) sets `_lifting`. The mist fades over `StageMeasure.veil` = **780ms** (`stage_tokens.dart:189`) with **`Curves.easeInCubic`** (`world_mist.dart:63`). That curve stays dark and then accelerates: at halfway, opacity is still about 0.88. The last ~150ms does most of the reveal. `onLifted` (`645-650`) removes the mist.

Haste (`587-593`, button “The road is known” at `1182-1185`) sets `_travellingTo` to null. No walk. Mist, then the room. That cut is the point of a known road.

### B. Enter, already here (`_enterScene`, `628-637`)

Mist on, wait for the painting and the faces, switch to the scene, lift 780ms `easeInCubic`. No walk. Tapping Enter does **not** auto-open a conversation. Travelling in does. Same place, two arrivals.

### C. The walk itself (`travel_transition.dart`)

`AnimationController`, duration **1200ms** default (`18`), **linear** (`forward()` with no curve, `30-33`). It does not repeat and it does not reverse. If the network is slower than 1.2s it sits on the last frame under the mist. If the network is faster, the parent deletes it mid-bob (`609-612`). `IgnorePointer` (`59`) so it cannot be skipped.

Per frame, `t` from 0 to 1 (`50-57`):

| Channel | Formula | What that is over 1.2s |
|---|---|---|
| Step phase | `t * 1.2 * 1.6 * 2π` | **1.6 steps/s**, about **1.9** head-bobs (`35`) |
| Bob | `sin(phase) * 3.5px * damp` | **3.5px**. `damp = 1 - easeInCubic(t)` (`36`, `53-54`). Still most of its size at halfway, gone at the end |
| Roll | `cos(phase) * 0.28° * damp` | **0.28°**. A shiver, not a gait (`37`, `55`) |
| Dolly | `1.06 - 0.06 * easeOutCubic(t)` | **6%** zoom, front-loaded. By ~480ms it is already ~1.3% from rest (`56`) |
| Image fade | `easeOut(min(1, t * 1.8))` | Opacity 0→1 over the first **~670ms** (`57`, `64-65`) |
| Vignette | radius `1.12 - 0.18 * sin(tπ)`, edge `0xAA` black, stops 0.58→1 | One breath (`88-95`) |

Underneath: flat `#0A0908`, then the painting or `#14100E` if the URL is null or still loading (`63-82`). `EverloreNetworkImage` then fades the bitmap in another **220ms** (`everlore_network_image.dart:43`) when it resolves.

The comment on the widget (`8-13`) says the last frame is held so the walk does not fade out while the road is still resolving. The hold is only the controller sitting at 1. The parent still cuts the whole overlay out in one frame, and the mist was covering it the entire time.

### D. What else is moving during that second

- Mist drift: a **9s** repeating controller (`world_mist.dart:23-26`). Five circles, `MaskFilter.blur(BlurStyle.normal, 48)` (`102`), repainted every frame for the whole wait, including the lift.
- On reveal, `SceneCast` is wrapped in `_FadeIn`: **420ms linear** opacity (`1570-1582`). Each figure with `arrive: true` (`world_characters.dart:161`) also runs a **560ms linear** slide from ±24% of its slot and a fade (`stage_figure.dart:62-65`, `127-139`; `StageMeasure.arrive` at `stage_tokens.dart:188`). Both start when the scene mounts, which is **before** the mist lifts. A 560ms entrance under a veil that takes 780ms to clear, and clears late, means the figure is already standing still when the player can see it.
- The story parchment fades on the same 420ms linear (`1383`).
- If a conversation was opened inside `_act`, none of that room entrance runs. The veil lifts onto `ConversationPanel` instead (`934-935`).

### E. Hitces, in the order they land

1. **Motion starts before the painting is decoded.** `setState` mounts the walk, then `awaitWorldFrame` (`587-600`). The first frames are the placeholder `#14100E`, then a 220ms image fade, while the bob clock is already running. Phase W2’s rule is the opposite: decode, then move.
2. **Two full-screen animations plus a radius-48 blur.** Mist and walk both rebuild every frame (`travel_transition.dart:47-48`, `world_mist.dart:60-62`). The blur is the expensive one. This is the disturbed part.
3. **The walk is removed and the room is built in one frame** (`609-612`), still under the mist. Backdrop, cut-outs, parchment, and title all mount together. Then the mist eases in (stays dark, then snaps).
4. **Cut-out warm and cut-out paint do not share a cache key.** `_awaitSceneFrame` precaches `CachedNetworkImageProvider` at `figureCacheHeight` computed from the **safe-area-shrunk** size and `figureSpeak` (`659-682`, `stage_tokens.dart:273-281`). `StageFigure` decodes at `(painted.height * dpr)` of the **full-screen slot after `roomChromeTop`** (`stage_figure.dart:232-239`, `281-286`). Those heights differ, so the face decodes again as the veil lifts. The handoff already recorded that a mismatched `maxHeight` warms a shelf nothing reads. The scene painting itself is safe: both the walk and `StageBackdrop` pass no `memCache` size (`travel_transition.dart:74-76`, `stage_backdrop.dart:41-43`), and the pre-travel warm matches that (`world_frame.dart:60-64`).
5. **Auto-opened conversation** (defect 2) replaces the room in that same arrival frame. The player never gets the place; they get the chat box.
6. **No route push during the walk.** A fight is a separate 420ms fade route (`405-414`) and only after a duel payload. It is not what makes arrival jerk. Leaving it out of this fix is right.

Enter-from-here has hitch 3, 4, and the snap lift, without the walk.

---

## 2. The room panel — inventory

There are two parchments. Arrival into a populated place usually shows the second one, not the first.

### 2a. The place (`_StoryPanel`, `1335-1489`)

Shown only when nobody is being addressed and no petition is waiting (`934-950`). A petition replaces this panel entirely (`859-861`, `1592-1703`). That replacement is deliberate.

Nesting, outside in:

1. `Align` bottom.
2. `Padding` 16 / 16 / `20 + safe bottom` (`1372-1376`).
3. `ConstrainedBox` **max 66%** of the screen (`1378-1381`, `roomPanelCeiling` `stage_tokens.dart:79`). No minimum. The box hugs its child until the cap, then scrolls.
4. `_FadeIn` 420ms linear.
5. `StagePanel`: gradient paper, 6px radius, brass hairline, shadow blur 26 (`stage_panel.dart:44-63`), grain stack, padding **20, 18, 20, 18** (overrides the token 18/16).
6. `SingleChildScrollView`, no controller, no keyboard dismiss, offset starts at **0** (`1386`).
7. `Column`, min size.

Contents, top to bottom. Sizes are the styles written, before `textScaler` (Flutter still scales `Text` unless told not to; the 66% cap does not).

| Order | What | Type | Container |
|---|---|---|---|
| 1 | `STR` `CHA` `LEAD` `LV` plus up to 3 standing tracks, if the save has them | caption, **10px**, tracking 1.1, brass, upper case (`1536-1566`) | `Wrap`, gap 8. Then 12px |
| 2 | Way-on, unless its kind is `choice` | Card: kicker **10px** tracking 1.5 (`SPEAK` / `WORK` / `THE ROAD` / `HERE`), label **14px** w700, blurb **12px** Garamond max 2 lines, arrow icon (`1023-1098`) | `Material` / `Ink`, radius 12, fill `0x33FFFFFF`, brass border. Then 12px |
| 3 | Headline | **Cinzel 18** w700 (`1399-1406`). `serifDisplay` is small-caps. This is a title face on what is often a sentence | plain `Text`. Then 8px |
| 4 | Body — the current beat | **Garamond 16**, height 1.5, ink muted (`1408-1414`). Token prose is 18 / 1.72 (`nexus_theme.dart:97-101`); this overrides both down | plain `Text` |
| 5 | One control per person standing | `StageChoice`, **min height 54**, dark gradient on the cream paper, brass rule, chevron, label **15px** Garamond w600 (`stage_choice.dart:54-84`). Copy is `Speak with {name}` or `… again` (`1416-1427`) | full-width button. Then 8px |
| 6 | Last reply, if any echoes | italic **13px**, max **2 lines**, ellipsis (`1428-1441`) | plain `Text` under that button |
| 7 | Drills, if any | kicker `WORK THE PLACE` **10px** tracking 1.4, then another 54px `StageChoice` each (`1446-1467`) | same button as talk and as choices |
| 8 | Each choice | optional `WEIGHT · …` or `HINGE · …` at **10px** (`1287-1309`), then the same 54px `StageChoice`, then an optional hint at **13px** (`1318-1328`) | `Column` per choice, 8px apart |

A room with traits, a way card, two people, and two choices is on the order of a stat row, a card, a small-caps headline, a paragraph, four 54px slabs, two echo snippets, and two hints, inside one scroll. Talk, work, and story choices are the same control. The beat is the quietest thing in the column. The darkest, tallest things are the buttons.

Scroll: one vertical scroller, top-anchored. There is no nested scroll inside this panel. A crowd of three or more that does not fit gets a **horizontal** `ListView` of figures behind the panel (`world_characters.dart:72-96`), not inside it. No keyboard here. If the column exceeds 66%, the player meets the meters and the headline; the choices are below the fold, with no hint that the paper continues.

The 66% cap and the inner scroll are the right idea (a petition comment at `1611-1614` says why). The failure is what is stacked, and that the scroll opens at the top.

### 2b. The conversation box (`ConversationPanel`, `world_characters.dart:169-667`)

This is the cluttered chat. It replaces the room panel (`934-935`) and paints its own backdrop (`441`).

Chrome above the paper, not in it: back button, the person’s name again in **Cinzel 18** centred (`499-509`). The name is then repeated on a plate.

The box:

1. `Padding` by the keyboard inset only (`591`). Scaffold `resizeToAvoidBottomInset` is **false** while talking (`701-703`), so the painting does not shrink. That part is right.
2. `SizedBox` height = `panelHeightFor(...) + safe bottom + 14` (`592-594`).
3. `panelHeightFor` is called with **`needed: 42% of the screen` and `composing: true` always** (`584-588`), not with the measured text and not with the composer flag. Floor 148, cap `min(screen - keys - 88, 48%)` (`stage_tokens.dart:199-210`). On a phone the paper is **always ~42% tall**, empty or not. `panelHeightMax` (26%, `stage_tokens.dart:120`) is never used here. The comment at `191-193` warns that using the max as a floor made an empty meeting a third of the screen; this call puts every meeting at 42%.
4. `StagePanel` `expand: true`, padding 18 / 22 / 18 / `10 + safe bottom` (`603-610`).
5. `Column`: `Expanded` transcript, then `AnimatedSize` **180ms `easeOutCubic`** footer (`623-625`).
6. Name plate, **Cinzel 13**, overlaps the top edge by 14px (`656-661`, `stage_name_plate.dart:62-67`).

Transcript (`705-746`): a `ListView` of **every** line this visit, plus echoed history once `_hydrateEchoes` has copied it (`interactive_world_screen.dart:273-278`). Each line is **Garamond 18**, height **1.55** (`757-760`). Older lines are the same size at opacity **0.72** (`731-732`). Gap 14. No collapse, no “earlier” control, no separate style for a past turn versus the current beat. Player and speaker differ by weight only.

How earlier words are reached: scroll that list upward by hand. The list is kept on the newest line by `jumpTo` (`366-383`), and `_settleAtBottom` may jump again for up to 8 frames (`386-394`). A growing line animates **220ms `easeOutCubic`** (`373-377`). `_followLatest` lets go only after the player is more than ~96px off the tail (`357-360`). There is no anchor to a collapsed current beat; the whole history is the scroll.

Footer, one of four, swapped by a `ValueKey` so it cuts rather than animates between them (`627-650`):

- Waiting: 14px italic “They hear you.” plus a 14px still (`1100-1130`).
- Idle: “Say something” in **Cinzel 14** (`803-829`).
- Revealing: `SizedBox.shrink`. The typewriter has no footer.
- Composing: a `0x33FFFFFF` field, brass border, a `**` marker button, hint “What do you say?” at 16px italic, field text **17px**, up to **4 lines**, and a filled “Speak” button (`917-985`, `1021-1055`).

Keyboard: the panel is padded up by `viewInsets` (`591`). `didChangeDependencies` jumps to the end when the inset changes if the field is open (`228-233`). The height formula **shrinks** the cap by the keyboard (`stage_tokens.dart:206`) and the padding moves the box as well, so the transcript is what gets crushed. History does not collapse first. Drag on the list dismisses the keyboard (`721`).

Typewriter (`321-354`): the first 8 characters appear at once, then every **18ms** a `setState` appends 8–36 characters and calls `_scrollToEnd`. Each tick rebuilds the whole `ConversationPanel`, backdrop and figure included (`422-537`). A long reply is a scroll fight, not a line being read.

Opening an existing thread does not type: `initState` shows the last non-player line in full (`217-220`).

### 2c. Events

Nothing in the room panel is an event inscription. What stands in for “something just happened”:

- Stat chips and the way card (always, not only when something changed).
- Hinge / weight kickers above a choice.
- The two-line echo under a Speak button.
- The scene headline and body, which **replace** the previous beat (`266-269`, `951-953`). Old beats are not kept as lines.
- The history icon opens `WorldMomentsSheet` over the whole screen (`986-993`). That is the road behind, not a line in the paper.

`StageInscription` (`stage_inscription.dart`) is the fight’s dark band. The room does not use it.

---

## 3. Ranked defects

Most visible first. Sizes are proposals for the fixer, not a contract change.

### 1. The player never sees a walk. They see fog, then a snap

`travel_transition.dart:59-98` under `world_mist.dart:64-76`, removed at `interactive_world_screen.dart:609-612`, revealed with `easeInCubic` over 780ms (`world_mist.dart:27-29`, `63`).

**Fix.** Decode the destination first (`awaitWorldFrame`), then play the walk **above** the mist. Crossfade the mist out over **600ms `easeOutCubic`** as the dolly starts, so the two are not stacked at full strength. Walk **2600ms**. Drive the dolly with **`easeInOutCubic`** from **1.08 → 1.0** across the whole duration (today’s `easeOutCubic` on 6% is finished by half a second). Bob **6px** at **2.0 steps/s**, damp `1 - Curves.easeIn.transform(t)` so it is still about half amplitude at 70% and zero at the end. Roll **0.4°**, same damp. If the bitmap is already decoded, start opacity at 1; do not also run the 220ms image fade. **Hold 400ms** at rest (zoom 1, bob 0). Then crossfade **500ms `easeInOut`** into the live `StageBackdrop` and fade the parchment in over **600ms `easeOutCubic`**. Do not unmount the walk in the same frame the scene mounts. Lift, if any veil remains, with **`easeInOutCubic` 700ms**, not `easeInCubic`.

### 2. Arriving walks straight into the chat box

`_shouldApproach` returns true for every `move`, `choose`, and `rule`, and also for any initiated line or finished fight (`452-456`). `_personToAddress` then picks the initiator or else **`_cast.first`** (`458-465`), and `_address` runs before the scene is shown (`375-377` inside `_act`, which `_travel` awaits at `603`).

A plain arrival should settle on the room. Auto-open only when `initiated != null` or a fight just ended — that is the “someone speaks about what just happened” case, and it should still land as one beat in the paper, not the full transcript. Who is allowed to speak is W5’s engine bug (`standing[0]`); this pass should not invent a new speaker rule. It should stop opening the transcript on an ordinary move.

### 3. The chat box is a 42% slab of every previous line at reading size

`world_characters.dart:584-588`, `718-745`, `757-760`.

**Fix.** Size the paper from the **current beat only**, clamped with `panelHeightFor(needed: measuredBeat, composing: _composing)` so an idle line can sit near the **26%** cap and the composer is what raises the ceiling toward **48%**. Current beat: Garamond **18**, height **1.45**, ink, max ~6 lines then this region scrolls. Everything older sits behind one control, collapsed (defect 4’s sketch). Drop the duplicate Cinzel name in the header (`499-509`); the plate already says it (`656-661`). “Say something” in Garamond **15**, not Cinzel **14**.

### 4. The room column gives the same 54px button to talk, work, and story choices, and parks the beat in between

`interactive_world_screen.dart:1391-1482`.

**Fix.** Order inside the same parchment: event lines, current beat, one “Earlier words” row, then choices. People are not a second choice list. One person: a single text row, Garamond **15**, “Speak with {name}”, min height **44**. Several: those rows only, no echo paragraph under each (the echo is the collapsed history). Keep `StageChoice` for story choices and drills. Move the meters out of this paper (they are a HUD; the map chrome or a single quiet line under the title can hold them). The way card becomes one inscription line, not a 3-line card.

### 5. Faces decode while the veil is lifting

Precache key: `interactive_world_screen.dart:667-682`. Paint key: `stage_figure.dart:281-286`. Different heights.

**Fix.** One function returns the `memCacheHeight` the slot will actually paint, and `awaitWorldFrame` uses that integer. Do not warm the safe-area size for a figure drawn on the full frame.

### 6. The mist’s blur is the hitch under the walk

`world_mist.dart:97-104`, five circles, blur **48**, every frame for up to 8s plus the lift.

**Fix.** Sigma **16**, or a static blurred asset. The 9s drift can stay. It must not run at blur 48 underneath a transforming full-screen image.

### 7. The typewriter rebuilds the room and yanks the scroll

`world_characters.dart:336-354` and `366-394`. 18ms `setState` on the whole panel; step of 8–36 characters.

**Fix.** If a reveal stays, tick every **32ms** by a word, and rebuild only the beat text (`ValueNotifier` on that string). One scroll to the end when the line starts and one when it finishes, **280ms `easeOutCubic`**. Do not jump on every tick. Prefer showing the finished line: the structural catch for a model that writes paragraphs is a calm paragraph, not a machine-gun append.

### 8. The walk’s own motion is a twitch, and it cannot be skipped

`travel_transition.dart:18`, `35-37`, `56`, `59`. 1200ms, 3.5px, 0.28°, dolly finished early, `IgnorePointer`.

**Fix.** The numbers in defect 1. Add a tap (the overlay, not `IgnorePointer`) that completes the hold and reveals. Known-road haste stays a short veil with **no** 2.6s walk.

### 9. When the room paper exceeds 66%, it opens on the meters, not the choice

`interactive_world_screen.dart:1378-1386`.

**Fix.** Keep the 66% cap. After layout, if `maxScrollExtent > 0`, jump so the **choice list’s top** is in view and the beat is what scrolls away upward — or, better, the collapsed layout in §4 fits without needing that. A 12px fade at the bottom edge when `extent > 0` is the only extra chrome.

### 10. The beat is Cinzel, and the body is smaller than the buttons

`interactive_world_screen.dart:1399-1414`.

**Fix.** Headline, if it is a short title, Cinzel **15** tracking 0.6, one line. Body, the thing to read: Garamond **18**, height **1.5**, `StageMeasure.ink` (not muted). Muted is for events and old lines.

### 11. Keyboard opens into a transcript that is still fully expanded

`world_characters.dart:580-591`, `1022-1025`.

**Fix.** Opening the field collapses history first (it should already be collapsed). Composer **min 1 / max 3** lines, not 4. Panel grows with the field up to the 48% ceiling; it does not keep a 42% history above the keys.

---

## Leave alone

- **The parchment.** Paper gradient, grain, brass edge, radius 6, the drop shadow (`stage_panel.dart:44-63`). The owner asked to keep the scroll container. Change what is inside it.
- **`StageChoice` for a real choice or a drill** (`stage_choice.dart`). Dark row, 54px min, brass rule, chevron, Garamond 15. It looks like a decision. It should stop being used as a talk button and as a person list.
- **Room veil** (`stage_tokens.dart:48-53`, `stage_backdrop.dart:23-36`). Only the bottom band is darkened. A fight-weight veil over a meeting was already found to swallow the speaker.
- **Where people stand.** `StageRoomLayout` rects, speak-rise, crop at the parchment (`stage_tokens.dart:596-694`, `world_characters.dart:39-116`). Do not go back to `Align` in a full-bleed `Stack`.
- **A petition replaces the scene copy** (`859-861`, `1592-1591`). Both claims in full, rulings as labels. Same 66% scroll is fine. Do not put chat history into it.
- **Visit lines are not a second memory.** Echoes come from the save (`273-292`). Collapsing them is presentation. Do not persist a parallel log on the client.
- **The history icon and `WorldMomentsSheet`.** That is the road behind, not “earlier words.”
- **Haste skips the long walk** (`593`, `1182-1185`). A known road is a short veil. Do not make it 2.6s.
- **`resizeToAvoidBottomInset: false` while talking** (`701-703`). Pad the paper, not the painting.
- **`panelHeightFor` refusing to use the max fraction as a floor** (`stage_tokens.dart:191-210`). The conversation call site ignores it; fix the call, not the helper.
- **The name plate** (one ribbon, Cinzel 13, flourishes). Keep it. Remove the second name in the conversation header.
- **“They hear you.”** (`1123-1130`). Quiet, in the world’s voice. Leave the copy.
- **Back from a conversation returns to the room** (`PopScope`, `998-1006`). Do not pop the route.
- **Place title in the top chrome** (`_Title`, Cinzel, with the paint shadow, `1706-1741`). Titles belong in Cinzel. The beat in the paper does not.
- **The 8s frame ceiling** (`world_frame.dart:14`). One dead image must not hold the curtain forever.
- **Map and fight.** Other tasks. The duel route’s 420ms fade is not this bug.
- **W2a’s words-on-the-veil**, if that task is in flight. This audit’s mist note is only the blur cost and the ease-in snap. Do not design a second veil.

---

## 4. Target layout

Same parchment, two contents. Arrival (defect 2) shows the room. Speak swaps the beat for the latest line and the choices for the say row. History stays one control, anchored under the current beat so the newest line never moves down when older lines open.

```
room, after the walk has held and crossfaded     ~26–40% of the screen
┌──────────────────────────────────────────────┐
│  The door was unbarred.                      │  13px italic, ink muted
│  A name you spoke is known here.             │  one line each, no card
│                                              │
│  The hall is quiet, and the fire has         │  Garamond 18 / 1.5
│  been laid for someone who is not you.       │  ink, the only reading block
│                                              │
│  Earlier words                          ▾    │  one row, Garamond 15
│                                              │  collapsed by default
│  ┃ Hear the watch                           │  StageChoice, 54px
│  ┃ Ask who laid the fire                    │  tidy list, 8px apart
│                                              │
│  Speak with the steward                      │  text row, 15px, not a slab
└──────────────────────────────────────────────┘
        16px gutter, 20px above the safe area
        cap 66%. If it still overflows, the
        choice list stays on screen and the
        beat is what scrolls upward.

expanded history (same paper, still anchored
so the latest beat does not jump):

│  Earlier words                          ▴    │
│  · You asked the way.                        │  13px muted, older
│  · She would not give it.                    │  max ~4 lines then
│                                              │  this region scrolls
│  She sets the cup down before she            │  18px, the newest, fixed
│  answers.                                    │  under the fold of the old
```

Open exchange, same paper, choices replaced:

```
┌──────────────────────────────────────────────┐
│  Earlier words                          ▾    │
│                                              │
│  She sets the cup down before she            │  18px current beat only
│  answers.                                    │
│                                              │
│  Say something                               │  15px, or the field
└──────────────────────────────────────────────┘
   name plate (Cinzel 13) overlapping the top
   edge by half the plate — already the case
```

Composer, when that row is tapped:

```
│  She sets the cup down before she            │
│  answers.                                    │
│  ┌────────────────────────────┬──────────┐   │
│  │ What do you say?           │  Speak   │   │  max 3 lines
│  └────────────────────────────┴──────────┘   │  paper grows, cap 48%
```

Events never get a fill, a border radius, or a chevron. One line, then they leave the paper on the next beat (the newest inscription can remain; a stack of more than three collapses into the same “Earlier words” affordance, under the spoken lines, still newest at the bottom of that region).

---

## Verification

No commands. The brief forbids edits outside this report and says `Servers: no`. Line numbers were taken from:

- `everlore/lib/features/interactive/presentation/travel_transition.dart`
- `everlore/lib/features/interactive/presentation/world_mist.dart`
- `everlore/lib/features/interactive/presentation/world_frame.dart`
- `everlore/lib/features/interactive/presentation/interactive_world_screen.dart`
- `everlore/lib/features/interactive/presentation/world_characters.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_tokens.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_panel.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_choice.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_figure.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_backdrop.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_inscription.dart`
- `everlore/lib/features/interactive/presentation/stage/stage_name_plate.dart`
- `everlore/lib/shared/widgets/everlore_network_image.dart`
- `everlore/lib/app/theme/nexus_theme.dart`

Not checked on the emulator: the exact ghosting of a 3.5px bob through `0xE6` mist, and the exact pixel gap between the two cut-out cache heights on `Pixel_10_Pro`. Both follow from the formulas; the device would only confirm the magnitude.

## Open questions

- W2a is adding words to this same mist. The walk should start only after that veil has handed off, or the title and the dolly will run together. Whoever writes the W2 brief should say which one owns the exit curve.
- After a fight, W4 wants someone speaking first. This audit keeps that auto-open and only removes it for a plain `move`. If the owner wants every arrival, including fights, to show the room panel with one new inscription instead, defect 2 should be tightened further. That is their call.
- Stat chips (`STR` / `CHA` / `LEAD` / `LV`) have no other home if they leave the paper. The fixer should not drop them on the floor. Under the place title, one muted line, is enough.

## Contract changes needed

None. Headline, body, choices, echoes, way-on, and spoken lines are already on the payload. Collapsing history and inscription styling are client layout. Who initiates speech stays the server’s `spoken.initiated`; the client should honour it instead of also opening `cast.first` on every move.
