# Phase 1 — Walks that play like a game

*Opened Oct 1 2026. Owner's five blockers from playing The Iron Verdict on the emulator. Orchestration per `rpg-ai/AGENTS.md`.*

The engine is sound (typecheck clean; `audit:iron-verdict` 30/30 places, 33/33 choices, 4/4 endings; all 60 assets reachable). What breaks the play is **presentation, pacing and continuity**: the map doesn't read as a game, entering a place is abrupt, the scroll panel is cluttered, fights look like two cards sliding, the player is left hanging after a fight, characters either know nothing or everything, and some rooms offer choices about people who aren't there.

Invariants every workstream keeps (AGENTS.md §3): the world is data; the server derives; the client renders and decides nothing; the model only writes words; outcomes are authored, never rolled; no hardcoded word lists.

---

## W1 — The map: markers, transitions, the place card  ·  UI (agy) + assets (Codex app)

**Today:** `world_map_view.dart` (532 lines) draws abstract markers over the stitched plates. States: `open`, `sealed`, `rumoured` (server `visibilityFor`, `world-play.ts:20`), plus *you are here*, *reachable in one move*, *the way on*, and fog. The place info box is the owner's "sloppy" card.

**Target:**
- A **marker system with one distinct, readable, premium look per state**: here / reachable / open-but-far / sealed / rumoured / way-on (quest lure). Idle motion for *here* and *way on* only; sealed and rumoured are still.
- Selecting a place: camera eases to it; the **place card** rises with the scene painting thumbnail, name, realm, a one-line description, who is known to be there, and one primary action (Travel / Enter / Sealed — with the in-world reason). Never more than one primary action.
- Travel: route draws, the camera follows, and it hands into W2's arrival.

**Asset decision (orchestrator's call):** the markers stay **code-drawn** — vector, shader and typography — not Codex sprites. The Sept 6 finding still holds: raster stickers fail at map scale (a carved stone marker read as an 11×18px smudge). Codex generates only **textures that frame the marker**: a brass/iron ring and seal plate, a wax-seal emblem for *sealed*, and a parchment card backing. All are flat, tileable and scale-safe. Batch: `.agents/assets/w1-map-chrome.md`.

**Design brief first** (orchestrator writes `everlore-docs/phases/phase-1-w1-map.md` with measured specs) → agy builds → I drive it on the emulator.

## W2 — Arrival and the room's scroll panel  ·  UI (agy)

**Today:** `travel_transition.dart` (103 lines) is the walking POV; the owner finds it abrupt and jerky. The room panel (`StagePanel` / `world_characters.dart`, 1132 lines) mixes story, choices, chat and events into one cluttered scroll.

**Target:**
- **Arrival walk:** slow, smooth and continuous (about 2.4–3 s). Eased dolly into the scene painting with gentle head-bob amplitude, fading in from the map, with no frame hitches. Precache gates it (the image must be decoded before motion starts). Skippable by tap.
- **The scroll stays** (the owner likes the parchment), but its content is re-laid out:
  - one clear current beat at reading size;
  - choices as a distinct, tidy list below it;
  - **conversation history collapsed** behind "earlier words" — expandable, scroll-anchored to newest;
  - events (gained, opened, heard) as quiet one-line inscriptions, not cards.
- Respects `textScaler`, small phones and the keyboard inset, with smooth expand/collapse.

## W3 — The fight  ·  UI (agy) + server contract (Codex) + assets (Codex app)

**Today:** `world_duel.dart` (461 lines) slides two portrait cards. Server `duel.service.ts` stages authored exchanges (actor, toll, bars, prose); the outcome is authored.

**Target flow:**
1. **Prepare:** dim arena painting; the player's fighter **small, bottom-left**; the opponent **small, top-right**. No panel on screen. A **shield emblem centre-screen** rotates and breathes (scale pulse, glint) — "ready yourself".
2. **Player's strike:** the shield pulses; tapping it makes the player's figure **lunge across to the opponent**, followed by the impact (flash, shake, spark burst, recoil, damage number, bar drain), then a return.
3. **Opponent's strike:** **no shield**. The opponent lunges at the player automatically, with a block or hit reaction on the player's side.
4. Prose from the server appears as a **brief caption strip**, not a panel, and never covers the fighters.
5. **Gamified vigour bars:** segmented, with a trailing "ghost" drain, a hit flash and a low-health pulse.
6. Verdict card → Continue → W4.

**Invariant:** the tap only paces the fight. **It never changes the outcome** (authored, or trait-contested on the server). The client decides nothing.

**Contract change (server):** each exchange must say **whose strike it is** (`attacker: 'player' | 'opponent'`) and how it lands (`landing: 'hit' | 'blocked' | 'grazed'`). These are derived from authored data, never from prose. → backend task.

**Assets (Codex app):** the shield emblem (brass and iron, front-on, on chroma green), an impact spark/slash sheet, and possibly the bar frames. The animation itself is code.

## W4 — Never left hanging; word travels  ·  backend (Codex) + UI (agy)

**Today** (`.agents/reports/r3-word-travels.md`, verified Oct 1):
- **"Witnessed" is inferred, not recorded.** `witnessedHere` (`world-play.ts:87–113`) feeds whoever stands in a room *now* the last four choices taken there, even if they weren't present, and cast presence is computed *after* a choice's flags apply, so someone the choice revealed "witnessed" it. No per-deed witness list exists.
- Cast has `faction` and `home_location_id` only. There are no kin or ally ties, and realm can be derived from home location.
- The client ignores `reign_opened` (`_apply`, `interactive_world_screen.dart:143–270`). The ending's opening beat is never shown.
- `wayOnOf` (`interactive-world.service.ts:288–358`) can return null, so a post-action room can have a panel but no speaker and no way on.

**Fix order:** (1) record the **pre-action cast** on each deed event as its witnesses, with structured `{deed_kind, deed_id, location_id, seq}` on the event (never parse `player_input`); (2) author ties + per-deed travel scope; (3) derive `heard` on read from the active event log (restore-safe), passed to `speakAs` as its own list alongside `witnessed` and `knowledge`; (4) client shows `reign_opened`; audit fails on any reachable state with no way on.

**Target:**
- After a duel or a quest/hinge completes, the client returns to the room with the panel up, **someone relevant speaking first** about what just happened, and the way on shown. There is never an empty room.
- **Word travels, structurally:** *witnessed* (present at the deed) knows now; *connected* people (kin / sworn / same household or faction — **authored fields on the cast**) hear after a delay measured in the player's own actions (moves or rulings); *unconnected* people never hear unless the player tells them. This is derived from the event log and authored links, so it is restore-safe. It is not stored per character and **never decided by the model**. `speakAs` is given only what that character has heard, tagged with how they heard it ("saw it" / "heard from X").

## W5 — Stale and phantom choices  ·  backend (Codex) + audit

**Today** (`.agents/reports/r2-stale-choices.md`, verified Oct 1). Symptom: an early room offered a choice about someone not there, and the tap opened a conversation with the room's own character.
- **It was never fixed, at any location.** The cause is an **engine** defect, introduced by `05939b2` and widened by `03a7ffe`. After any `choose`, `speakerAfter()` falls back to `standing[0]` (`interactive-world.service.ts:288–304, 1315–1356`), and the client opens that person's conversation. A choice has no structural target (`WorldChoice`, `world-source.ts:60–99`), and `choiceOffered()` (`world-play.ts:153–171`) never checks presence.
- The owner's case is almost certainly **`trust_cassian`**: "Go to Cassian", offered at `verdict_ring`, though Cassian lives at `chainhouse` and is hidden once `cassian_dead` is set.
- `ask_the_stair` stays offered after `cassian_dead`. `fight_succession` ("in Cassian's place") lacks `not_for_leads: cassian_vale`.
- Spent choices and `restore` are correct.

**Fix:** (1) **engine:** optional authored `character_id` on a direct-address choice; offered only if `presentCast()` returns that person and they aren't the lead; after the choice, the target speaks — **never an unrelated `standing[0]`**; (2) **data:** the three fixture fixes above; (3) **audit:** fail on a target missing from the cast, absent under the choice's own offer conditions, or equal to the lead. Prove it bites with an invalid target and a flag-hidden target. Never infer targets by scanning labels for names.

**Target:** an **engine-wide** fix, not a per-location patch. A choice that involves a person carries that person's **cast id** (authored, structural). It is offered only when that person is present and alive, isn't the lead, and the choice hasn't been spent. The audit fails on any choice whose involved person cannot be present when it is offered.

---

## Order

```
now        r2 + r3 readers (investigation)               ← running Oct 1
next       W5 backend (engine fix + audit)               Codex
           W3 contract (attacker/landing)                cmd   (parallel, different files)
           W1 design brief + W1/W3 asset batches         orchestrator + Codex app
then       W1 map UI, W3 fight UI, W2 arrival+panel      agy, one at a time (same feature folder)
           W4 backend (word travels)                     Codex
last       W4 client ("never hanging")                   agy
each       orchestrator drives it on the emulator, then shuts servers down
```

The three UI workstreams all live in `everlore/lib/features/interactive/presentation/`. They run **sequentially** (or are split by file with non-overlapping `Paths:`); `interactive_world_screen.dart` belongs to one task at a time.

## Pending, carried over

- Uncommitted on disk: training no longer prompts a character to speak (server + app), and the seed always rewrites the fixture. Verify on the emulator, then commit.
- The Iron Verdict is seeded **unpublished**, so it's not on the Walks shelf. Owner to decide: publish to all, or only to their account.
