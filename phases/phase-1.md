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

**Contract change (server) — W3-C, fixed Oct 1.** Every field is derived in `duel.service.ts` from authored data and the bound lead. None comes from prose or the model, and existing fields are unchanged (additive only).

```ts
// StagedDuel / OfferedDuel — added
player_side: 'challenger' | 'defender' | null
  // the side whose combatant is_player, or whose cast_id === the bound lead.
  // null = a Verdict the player only watches (no shield at all).
near_side: 'challenger' | 'defender'
  // drawn small, bottom-left. = player_side ?? 'challenger'. The other side is top-right.

// StagedBeat — added
striker: 'near' | 'far'
  // which corner lunges this exchange: actor === near_side ? 'near' : 'far'
awaits_player: boolean
  // true iff player_side !== null && actor === player_side.
  // The client shows the pulsing shield and waits for a tap. Never true for the opponent.
impact: 'blocked' | 'glancing' | 'solid' | 'decisive'
  // decisive  iff the beat is the decisive one
  // blocked   iff toll === 0
  // glancing  iff toll * 4 < duel vigour   (under a quarter of a full bar)
  // solid     otherwise
```

Client rule: a tap on the shield releases the next `awaits_player` beat and nothing else. It never picks, skips or changes an outcome. Spectator Verdicts (`player_side === null`) play on their own with a tap-to-advance at most.

**Assets (Codex app):** the shield emblem (brass and iron, front-on, on chroma green), an impact spark/slash sheet, and possibly the bar frames. The animation itself is code.

## W4 — Never left hanging; word travels  ·  backend (Codex) + UI (agy)

**Today** (`.agents/reports/r3-word-travels.md`, verified Oct 1):
- **"Witnessed" is inferred, not recorded.** `witnessedHere` (`world-play.ts:87–113`) feeds whoever stands in a room *now* the last four choices taken there, even if they weren't present, and cast presence is computed *after* a choice's flags apply, so someone the choice revealed "witnessed" it. No per-deed witness list exists.
- Cast has `faction` and `home_location_id` only. There are no kin or ally ties, and realm can be derived from home location.
- The client ignores `reign_opened` (`_apply`, `interactive_world_screen.dart:143–270`). The ending's opening beat is never shown.
- `wayOnOf` (`interactive-world.service.ts:288–358`) can return null, so a post-action room can have a panel but no speaker and no way on.

**Contract — W4-S, fixed Oct 1** (server; the client is unchanged except for showing `reign_opened`):

```ts
// Authored, optional, on a cast member
ties?: string[]            // cast ids this person is close to (kin, sworn, household). Read SYMMETRICALLY.
// Authored, optional, on a choice (and a petition resolution)
word?: 'private' | 'circle' | 'public'   // default 'circle'
// World-level, authored, optional
word_travel?: { circle_after: number, public_after: number }   // default { circle_after: 2, public_after: 4 }

// Stored on each material event (choose / rule / duel), never parsed from player_input
deed: { kind: 'choose' | 'rule', id: string, location_id: string, witnesses: string[] /* cast ids */ }
  // witnesses = presentCast at that location BEFORE the action applied (pre-action),
  //             so someone the deed itself revealed did NOT witness it.

// Derived on read, never stored, from the ACTIVE event log (restore-safe), per cast member C, per deed D:
//   saw    if C ∈ D.witnesses
//   heard  if word ≠ private and C shares a faction (≠ 'none') or a tie with some witness W,
//          and ≥ circle_after material actions have happened since D        → from: W
//   heard  if word = public and C's home realm = D's location realm,
//          and ≥ public_after material actions since D                       → from: 'word on the road'
//   else   unknown
```

`speakAs` receives `witnessed` (deeds C saw) and `heard` (`{ text, from }`) as **separate lists**, built from the above. This replaces `witnessedHere`'s location-based inference. "Material actions" = choose + rule + move; talk and train don't advance word.

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

## W2a — The veil: words while the world loads  ·  UI (agy)   *(owner's request, Oct 1)*

Every wait on images (opening a Walk, walking into a place, the fight's opening) shows today's `WorldMist` alone, which reads as a blank screen for up to 8s. Replace it with one shared **`WorldVeil`**: the mist plus **animated text in the world's own voice**.

- **Words come from data the client already has, never from client copy:** opening a Walk shows the chapter title, then the world's `blurb`. Walking into a place shows the place's name, then its first description sentence. A fight shows the Herald's question. The client never names a world. With no data, show the mist alone.
- **Motion:** the title fades and rises (12px → 0, 700ms, `easeOutCubic`) in `serifDisplay` (Cinzel, titles only), letter-spaced. The line below reveals word by word (each word fading in over 260ms, staggered 90ms) in `aiText` (EB Garamond) at 70% parchment. A thin brass rule draws outward from the centre beneath the title (500ms). After the full line shows, a slow breathing opacity (0.75 ↔ 1, 2.4s) on the line, so a long wait still feels alive.
- **Exit:** when the frame is ready, text and mist lift together (the existing `lifting`/`onLifted` path). If the wait is shorter than the title's entrance, still show the title for ≥ 600ms so it never flickers.
- Respects `disableAnimations` (static text, no stagger) and `textScaler`; centred; max 3 lines; never covers the safe area.

---

## W6 — Every screen says what to do next  ·  backend (Codex) + data (cmd) + UI (agy, Grok)   *(owner's play-test, Oct 2)*

The owner played the Iron Verdict and lost the thread: after the fight the Herald spoke one long paragraph and the only control was "Say something"; meeting Nara Voss, the same. The room stacked four echo lines over the scene, the title and stats sat on the figure's head, and Cassian's face changed between exchanges in a fight.

### Contract

**Cast data** (`iron-verdict.cast.json`, authored):

```jsonc
{
  "id": "nara_voss",
  "pronouns": "she",                 // "she" | "he" | "they" — matches the painted portrait
  "topics": [                        // 2–3 things the player can ask this person
    { "id": "nara_debt", "ask": "Who holds the debt now?", "requires": ["verdict_witnessed"], "forbids": [] }
  ]
}
```

- `ask` is what the player says: first person, under 60 characters, no flag or id words.
- A topic is offered while its `requires`/`forbids` hold and it has not been asked in this save.

**Presence** (`present[]` in the state payload) gains `topics: [{ id, ask }]` — only the open, unasked ones. Asked topics are part of the conversation record, so a rewind past the asking makes the topic open again.

**Talk** — `act({ type: 'talk', character_id, topic_id })` is accepted as well as `said`. The server sets `said` to the topic's `ask`; a topic that isn't offered right now gets a 400 with an in-world message. The query budget for talk stays at 9.

**Spoken** (`spoken` in the act response, and each `echoes[].replied`) gains an optional `beat`:

```jsonc
{ "character_id": "the_herald", "name": "The Herald", "line": "The debt stands. It passes to the house that buried him.", "beat": "The Herald lowers the trumpet.", "portrait_url": "…" }
```

- `line` is only the words the character says aloud, at most 3 sentences.
- `beat` is at most one sentence of what they do, or absent.
- The split is made **structurally** from the model's text, not asked for by instruction: quoted spans become `line`, and the first non-quoted sentence becomes `beat`. Text with no quotes is treated as speech.
- A `beat` whose pronoun disagrees with the speaker's authored `pronouns` is dropped.
- The speaker's `pronouns` are passed into `SpeechContext.situation`.

**Fight** — a fighter wears **one face for the whole fight**: their default portrait, the same one the room shows. `beats[].portrait_url` equals the fighter's `portrait_url`. Bearings stay in use outside fights.

**`way_on`** is unchanged, but it is now shown in the conversation as well as in the room.

### UI

- **Room header:** the kicker and title only. Strength, Charisma, Leadership and the standings move into a collapsible strip, closed by default, like the chat's Realm Status. It never covers the figure.
- **Figure** (room and conversation):
  - It sits in the zone from the header's bottom (plus the open strip, if any) + 12px down to the panel.
  - It is scaled to fit that zone, with its feet behind the panel's top edge (submerged by about 10% of its height).
  - It is never taller than 60% of the screen and never touches the safe area.
- **Room panel:**
  - At most **one** echo line (the newest).
  - The scene title and its prose.
  - The way on, marked "Next".
  - The other choices.
  - "Earlier words" opens a sheet; it is never expanded inline.
- **Conversation:**
  - The `beat` is small and italic, and the `line` is the speech.
  - Under it: the topics, then the way on ("Next · …", which closes the conversation and follows it), then "Say something" and "Step away".
  - The first meeting shows the topics too.
- **Advance mark:** the pulsing chevron points right (›) everywhere — fights, conversation, prologue, overture.

## W7 — A character answers like a chat turn, streamed  ·  backend (Codex) + UI (agy)   *(owner, Oct 3)*

**Why.** Walkable speech was a blocking structured-JSON call inside `act()` (2.4–4.5 s before the room, the echo or Next appeared) with a brief that sent the NPC's own past replies as the player's turns and gave no player turn before an NPC spoke first — so the model wrote two-person scenes and spoke as the lead. Chat worlds never have this: the player's message is the user turn, the character is the assistant, and the reply streams. The walkable world now does the same. Map: `.agents/reports/r6-speech-stream-map.md`.

**Decisions (owner).** (1) A character's secret (`reveals_flag`) is authored to a **topic**: asking that topic, when its gates hold, sets the flag in `act()` itself; the model only voices the answer — no model judgement moves story state. (2) Words already on screen are never taken back (chat's rule): a streamed line that fails a post-stream check stays visible, is logged, and is NOT written into the character's conversation history.

### Data
- A cast topic may carry `"reveals": true`. At most one per cast member; only on a member with `reveals_flag`. Asking it (`act({type:'talk', topic_id})`) with its `requires`/`forbids` satisfied sets that member's `reveals_flag` through `worldVocabulary()` exactly as the old `turned` path did. Topic consumption (`asked_topic_ids`) is recorded in `act()` when asked — no longer contingent on a reply.
- Model-driven `turned`, `disposition_delta` and `bearing` are removed. Disposition stays at its authored value (context only; nothing gates on it). The speaker shows their default portrait (one face per conversation, as with fighters).

### `act()` — returns at once
- No model call inside `act()` for speech. State, echo, `way_on`, cast, progression, the reveal flag (if any) and the event are committed and returned immediately.
- When someone will speak (the talk target; or the existing post-action speaker rule — `speakerAfter` / `addressCue` conditions), the response carries
  `speech: { id: string, character_id: string, name: string, portrait_url: string | null }`
  and the server starts the stream after responding. `spoken` is no longer returned by `act()`.
- `state.speaking = { id, character_id, started_at }` while a stream is live (omit the key when idle — never null).

### The stream
- Prompt, chat-shaped: **system** = the character card (name, role, voice, wants, fears, knowledge via `knowledgeFor`, witnessed/heard deeds with the doer named, where they are, who they're speaking to, the world's voice). **messages** alternate: `user` = what the player said, or the player's deed rendered as a chat action line, e.g. `*Cassian Vale swears a challenge on the sand.*` (via `deedForWitness`); `assistant` = the character's own earlier replies verbatim. Format asked for: chat style — spoken words in quotes, one short action in `*asterisks*` at most, 1–3 sentences.
- Model: the chat narrator's streaming path (`callLLMStreamWithFallback`, the world's SFW/NSFW narrator model and fallbacks, `sessionId: instanceId`, first-token/idle timers). Configurable via `CHARACTER_SPEECH_MODEL` for comparison.
- Frames on `user:<playerId>:events` → `/ws/play` (existing relay; the client filters by `instanceId` and `speechId`):
  - `world_speech_started { instanceId, speechId, characterId }`
  - `world_speech_delta { instanceId, speechId, delta }`
  - `world_speech_end { instanceId, speechId, line, beat? }` — `line` = the words, `beat` = the `*action*` if it passes the gesture rules
  - `world_speech_failed { instanceId, speechId, fallback? }` — only before any delta; `fallback` = `cast_unanswered` for direct talk, absent for an initiated address
- After the stream: split words/action, run the speech checks (narration, self-address, lead voice, repeat, grounding). Pass → append `{said, replied, beat?}` to `conversations[character_id]` (version-guarded write). Fail → keep visible, log `world_speech.flagged { reason }`, append nothing. Either way clear `state.speaking`.
- No retries once a delta has been sent. Before the first delta, a provider error may fall back to the next model (chat's rule).
- Where it runs: in the API process right after `act()` responds (no queue hop, so first token is the model's own TTFT), with an in-memory `instanceId → AbortController` registry for cancellation, publishing through the existing Redis user channel. If the API restarts mid-stream, the stream is lost: `state.speaking` older than 60 s is treated as idle and cleared on the next read.

### Concurrency, rewind, reconnect
- Any `act()` or restore while `state.speaking` is set cancels that stream: abort the request, send `world_speech_end` with whatever was already sent (or `world_speech_failed` if nothing was), commit nothing to history. Late frames for a cancelled `speechId` are ignored by the client.
- On reconnect the client drops any half-shown stream and refetches state (history shows the line if it committed).

### Client
- Apply `act()` immediately (room, echo, Next). If `speech` is present: show the speaker (default portrait) with a speaking indicator, render deltas incrementally (chat's buffered cadence), finalise on `end`, show `fallback` or remove the indicator on `failed`. The player can act at any time; a new act cancels the stream.
- `WsManager` gains the four frame types; the interactive screen ensures the socket is connected while open.

### Audits
- `audit:act-queries`: talk/move/choose/rule make no model call and stay within budget.
- New `audit:world-speech`: prompt roles (no NPC reply ever in a `user` turn; a deed is a `user` turn), frame order, cancel-on-act, cancel-on-restore, flagged line not in history, reveal-by-topic sets the flag only when gated, no reveal without a topic.
- `audit:iron-verdict`: every `reveals_flag` is reachable through its reveal topic or an authored choice on some road.

---

## W8 — Fight bars that count; the Verdict ships published; seen live  ·  backend (cmd, Codex, Grok) + UI (agy)   *(owner, Oct 3)*

The owner wants the fight's bars **accurate**: real numbers on screen, never invented ones.

### Contract — fighters carry their real stats

- Every staged fighter (`StagedFighter`, and the wire `duel.challenger/defender`) gains `level: number` and `strength: number`.
  - The **bound lead**: their *live* `state.traits` (after training).
  - A **cast member**: their authored `start_traits`.
  - A **duel-only combatant** (no `cast_id`): authored `traits: {strength, level}` on that combatant in the world file. Validation **rejects** a duel combatant without them — no defaults, no client fallback.
- `vigour` (start) and per-beat `challenger_vigour`/`defender_vigour` are already the real authored arithmetic (start − authored tolls). Unchanged.
- Outcomes stay authored. Stats are shown, never rolled.

### Client

- Each status strip shows the fighter's name, `Lv <level>` and `Str <strength>`, and the bar shows its count: `<current> / <vigour>`, updating beat by beat with the bar. Numbers in tabular figures, no other numbers anywhere.

### Publishing

- The world file may declare `"published": true`. `upsertWorldFromAuthored` then sets `is_published: true` on every upsert (not only on insert); a file without it leaves the stored flag alone. The Iron Verdict declares it, so a deploy's seed publishes it. Nobody writes prod by hand.

### Seen live (no new contract)

A capped live coverage run drives, through the local API as a throwaway player: (a) reign petitions ruled until a **ripened** petition appears and is ruled; (b) the `let_cassian_throw` and `fight_succession` duels end to end; (c) an act sent **mid-stream** — the old line ends/fails, its history is not written, the new speech streams.

---

## W10 — Know what to do; earn it; read it plainly  ·  backend (Codex) + data (cmd) + UI (agy)   *(owner's play-test, Oct 4)*

The owner played the Verdict and could not follow it:
- the Herald was handed the *outcome text* of a choice as if the player had said it, repeated it back, then told the player "You said the debt stands";
- "Earlier words" printed walks and choices as "You: *…*";
- the Herald pointed to the Iron Fane while the Next card pointed to the Chainhouse;
- the prose is too dense to follow, the conversation box is cluttered, the player's own portrait vanished once the other side spoke, and nothing is ever *earned*.

The approved mockup takes its goals from Assassin's Creed. There is **one tracked goal**, always top-left: quest name, goal, where. Goal complete and New goal get centred moments. A goals journal opens from the tracker. Achievements arrive as a slide-up card and are listed beside the walks.

### 1. Story fixes (server)

- **A deed is not speech.** A conversation exchange is stored as:
  - `{ said, replied, beat? }` — the player typed or tapped words;
  - `{ did, replied, beat? }` — the player walked, chose or ruled.

  Old records whose `said` is wholly wrapped in `*…*` read as `did`.
- **The deed sent to a character is what the lead *did*,** not what came of it: the choice's `label`, or the resolution's `label` for a ruling, recast with the lead named (*Cassian Vale chose to stand by the verdict*). Never `echo`, `summary` or a duel verdict — those are outcomes, often in the world's own voice. A walk is "<lead> arrived at <place>."
  - The deed turn is framed as an event, not as words the lead spoke.
  - A character may answer it, or say nothing new.
- **No echoing.** A reply that substantially restates the deed or the scene prose is flagged with the same structural check as `nearPrevious`, and is kept out of history.
- **Characters point where the tracker points.** The speech context gains one situation line naming the tracked goal and its place. If asked what to do next, the character can answer the same way.
- **Plain words.** The speech brief asks for plain, modern, everyday English and short sentences. The world's own prose is rewritten plain in data (w10-plain).

### 2. Goals — data `<world>.goals.json`

```jsonc
{
  "quests": [ { "id": "the_sealed_verdict", "title": "The sealed verdict" } ],
  "goals": [
    {
      "id": "gaunts_key",
      "quest": "the_sealed_verdict",
      "text": "Ask Gaunt what the Duke's unused key opens",          // ≤ 60 chars, imperative, plain
      "why":  "The Duke had a key nobody used. Gaunt keeps every key in the Chainhouse.", // ≤ 160 chars, 1–2 sentences
      "by":   ["gaunts_key"],                                       // choice ids
      "warn": "Destroys your only proof."                           // optional, ≤ 60 — for choices that close doors
    },
    { "id": "hear_petitions", "quest": "the_reign", "text": "Hear the petitions waiting in the duchy",
      "why": "…", "petitions": true }                               // instead of `by`
  ]
}
```

- Every choice id belongs to **at most one** goal.
- Every choice reachable on some road (the `audit:iron-verdict` walk) belongs to **exactly one**.
- `quest` names a quest in the file, and quests are listed in story order.
- No flag, id or mechanic words in `text`, `why` or `warn`.

**Derived on every read, never persisted:**
- A `by` goal is **open** while any of its choices is offered at a place the player can reach. Reachability uses the same rules as `wayOnOf`: unlocked, open, along travel edges. The goal's `at` is the nearest such place and `roads` is the number of moves to get there (0 means here).
- A `by` goal is **done** once any of its choices has been taken. A `petitions` goal is open while there are unruled petitions, and is never done.
- **Tracked** is the first open goal in this order:
  1. no `warn` before `warn`;
  2. **what the last deed opened**: the latest position in `taken_choice_ids` of a taken choice whose `sets` includes a flag that the goal's choices `require`;
  3. `roads`;
  4. file order.

  Leads use the same order. *(Revised after the live run: nearest-first pointed away from where the story had just turned.)*
- `way_on` now points at the tracked goal: its first step, or the choice itself when the goal is here. The map cue and the tracker never disagree.

**Payload.** State and `act()` responses gain `goals`:

```jsonc
"goals": {
  "tracked": { "id", "quest_title", "text", "why", "at", "at_title", "roads", "warn"? } | null,
  "leads":   [ /* same shape, other open goals, nearest first, ≤ 6 */ ],
  "done":    [ { "id", "quest_title", "text" } ]            // in the order the choices were taken
}
```

`act()` also returns `goals_completed: [{ id, quest_title, text }]`: the goals that are done after this act and were not done before it. It is empty on restore or rewind.

### 3. Achievements — data `<world>.achievements.json`

```jsonc
[ {
  "id": "ink_before_blood",
  "title": "Ink before blood",                    // ≤ 28 chars
  "blurb": "Prove the verdict was written before the fight",   // ≤ 70 chars
  "icon": "award",                                // award | gavel | sword | scroll | key | map | crown | skull | shield | eye | coin | flame
  "hidden": false,                                // true ⇒ locked shows as "???" (endings, twists)
  "when": { "flags": ["…"] }                      // ALL keys present must hold
} ]
```

- `when` keys:
  - `flags`: all of these authored flags are true;
  - `any_taken`: at least one of these choice ids has been taken;
  - `ending`: an ending id, or `"any"`;
  - `rulings_at_least`: ledger length is at least this;
  - `visited_at_least`: unlocked places are at least this;
  - `goals_done_at_least`: done goals are at least this.
- There are 12–20 achievements per world, and every one must be reachable on some road (audit).

**Persisted — this is a record of the player, not world state.**
- Achievements are stored in the collection `interactive_achievements` as `{ user_id, world_key, achievement_id, instance_id, unlocked_at }`, with a **unique** index on `(user_id, world_key, achievement_id)`.
- `act()` checks them in its tail, **after** the version-guarded state write succeeds. A duplicate insert is ignored.
- Rewind never takes one away.
- Account deletion purges them, at **all three** purge sites in `deletion.service.ts`.
- `act()` returns `achievements_unlocked: [{ id, title, blurb, icon }]`. Any `act()`, including the first one after a deploy, may unlock several at once.
- `GET /interactive-worlds/achievements` is authed and returns only the caller's own records:

  ```jsonc
  [ { "world_key", "world_title", "unlocked": 4, "total": 18,
      "items": [ { "id", "title", "blurb", "icon", "unlocked_at" }          // unlocked
               , { "id", "title", "blurb", "icon", "locked": true }         // locked, not hidden
               , { "id", "locked": true, "hidden": true } ] } ]            // hidden + locked: NOTHING else sent
  ```

  It lists the worlds the player has a walk in or an achievement from. Unlocked items come first, newest first.

### 4. Client (agy) — the approved mockup

- **Tracker.**
  - It sits top-left on the room and conversation screens, under the back arrow and clear of the figure.
  - It shows the quest title in Cinzel small caps (10–11 pt, brass), a diamond marker with the goal `text` (EB Garamond 16), and `at_title · N roads away`. When `roads == 0` the last line reads "Here".
  - Tapping it opens the **goals journal**.
  - It is hidden in fights, the prologue, the overture and endings, and when `tracked` is null.
- **Goals journal** (a sheet):
  - "Tracking": the goal, its `why` and its `warn`;
  - "Other leads": tapping one pins it, so the tracker shows the pinned goal while it stays open. The pin is a client-local preference per save, not world state;
  - "Done": a checklist.
- **Goal complete / New goal.** On `goals_completed`, a centred moment with thin brass rules: "GOAL COMPLETE" + the goal, about 2 s; then "NEW GOAL" + the new tracked goal. It never stacks on top of a fight or the prologue — it waits until they close.
- **Achievement card.**
  - On `achievements_unlocked`, it slides up from the bottom and fades after about 3 s.
  - Several arriving together queue one after another. After 3, a single "+N more achievements" card replaces the rest.
- **Achievements tab.**
  - The clock-icon screen (`my_walks_screen.dart`) gets two tabs, **Walks | Achievements**.
  - Achievements are grouped per world, with an "N of M" bar and a grid of round badges.
  - A locked badge has a dashed outline and a lock, and shows its title. A hidden one shows "???".
- **Conversation, decluttered.**
  - Both figures (the speaker and the lead) stay on screen for the whole conversation. The lead never vanishes when the other side starts speaking.
  - The panel shows, in order:
    - the speaker's name (Cinzel kicker);
    - **your last words** as one small muted line ("You asked: …"), or nothing if the last turn was a deed;
    - the reply (the only large text);
    - the topics as wrap chips;
    - one row with the "Say something" field plus two icon buttons: Earlier words (a history icon that opens a sheet) and Step away (a door icon).
  - **No Next card** — the tracker replaces it. No inline history.
- **Earlier words sheet** shows only `said` turns ("You: …") and replies. `did` turns are not shown.
- **Room panel:** the Next card is removed there too (the tracker replaces it). The scene prose and the choices stay.

### Audits

- `audit:iron-verdict`:
  - goal coverage (every reachable choice is in exactly one goal);
  - tracked is never null before an ending, along every road;
  - `way_on` agrees with the tracked goal;
  - every achievement is reachable on some road.
- `audit:world-speech`:
  - a deed turn never carries `echo`, `summary` or verdict text;
  - a `did` exchange is never presented as player words;
  - an echoing reply is flagged and kept out of history.
- A new achievements check, folded into `audit:iron-verdict` or a new audit:
  - a unique-index duplicate is ignored;
  - a hidden locked item serialises to `{id, locked, hidden}` only;
  - rewind keeps unlocks;
  - deletion purges at all three sites.

### W10a — Characters follow the pinned goal  *(owner, Oct 4)*

The pin stays a client preference: it is never stored on the server. The app sends it with each action, and the server only reads it.

- `act()` accepts an optional `pinned_goal_id: string` (1–80 chars, route-validated) on every action type.
- The server derives open goals as usual. If `pinned_goal_id` names a goal that is **open right now** (tracked, or one of the leads), the speech context's `trackedGoal` is that goal. Otherwise it is the derived tracked goal. An unknown, done or unreachable pin is ignored silently.
- The pin never changes the payload's `goals.tracked`, `leads`, `way_on`, `goals_completed` or achievements. It only changes what a speaking character is told.
- **Client:** every `act()` call passes the pinned goal id for this save from `GoalPinStore`, when there is one.

---

## W11 — Every bug from the full playthrough  ·  backend (Codex) + data (cmd) + UI (agy)   *(owner: "fix all of it", Oct 4)*

The source is `playtests/2026-10-04-iron-verdict-full-run.md`. Every item there is in scope, and the numbers below refer to the sections of that file.

### Server contract (Codex)

1. **A lead's place is always theirs.**
   - The `world.locations[].visibility` the player is sent is `open` for the lead's current location, whatever its seal.
   - **New `state.visited_location_ids: string[]`**: the places the lead has actually stood in. `bind`/`begin` adds the start place, and every `move` appends. It is snapshotted with checkpoints, so a rewind restores it. Absent on old saves means `[current_location_id]`.
   - The client opens the map on `current_location_id`.
2. **Goals.**
   - A `by` goal's `at` is the place offering one of its choices that **the most recent deed opened**: a place whose offered choice `requires` a flag set by the latest taken choice. Failing that, the nearest place, as before.
   - **After an ending** (`progression.ending` set), pre-ending `by` goals are no longer offered: tracked is the `petitions` goal while it is open, and `leads` is empty. `goals_completed` never reports a goal on the act that produced an ending; the ending is the moment.
3. **Achievements, one system.**
   - The older `progression.marks` are **removed from the payload** and from the ending view. Achievements are the only record.
   - The ending screen shows the achievements unlocked by this walk. `progression.ending_achievements: [{id,title,blurb,icon}]` lists the ones whose record has this `instance_id`.
   - `when.visited_at_least` counts `visited_location_ids`, not unlocked places.
4. **Who speaks after a deed.**
   - After `choose`/`rule`/`train`, a character speaks only if the deed names them (`choice.character_id`), or the deed was a fight they witnessed (current duel rule).
   - After `move`, someone speaks only on a **first meeting** at that place.
   - Otherwise there is no `speech`. A talk the player starts is unchanged.
5. **No silent speech.**
   - Every speech job ends in exactly one logged terminal line: `world_speech.end`, `.flagged`, `.failed` or `.cancelled`, each with a reason.
   - Investigate why a wordless "Speak with … again" talk produced no frames, then fix it. It must either speak, or end with a fallback line the player sees.
6. **Echo check sources:** the deed label, the deed's `echo` and `summary` (or the resolution's `echo` and `consequence` for a ruling), and the scene body.
7. **The lead is never a party to a petition.** Petition parties may carry `cast_id` (data). A petition with a party whose `cast_id` is the lead is not offered to that lead.
8. **Ruling outcome:** the `rule` act response gains `ruling: { title, label, consequence_shown }`. `consequence_shown` is the resolution's `echo`, never `consequence`. The client shows it as a moment.
9. **Walk deletion** skips vector-store cleanup for interactive instances, and a 404 there counts as already gone. No error log.
10. **Walk list:** each walk carries `lead_name` and `place_title` (`GET /interactive-worlds/instances` and the "wandered here before" listing).

### Data (cmd)

- **Scene text written for one lead:**
  - every `scene_body`/`scene_when` variant and every echo or summary that describes a lead in the third person gets a lead-neutral rewrite or a per-lead variant (`for_leads`/`not_for_leads`);
  - add post-Verdict and post-ending variants where the scene still describes the time before;
  - place `description`s must be timeless (no "A new clue waits").
- **`hear_the_ring`** gains `forbids: [verdict_witnessed]`. Check every lead's opening choice for the same leak.
- **Petition parties** that are cast members get `cast_id`.
- **Achievement copy** is lead-neutral. "Seven names on the road" now means places *visited*: keep 7 and reword it ("Stood in seven places").
- **The marks block** in `.progression.json` is deleted. Each mark worth keeping becomes an achievement.
- **Endings:** each `reign_description` is ≤ 60 words, and each ending gets a one-line `reign_hint` explaining its reign verb ("Reform: hear the duchy's petitions and remake its law").

### UI (agy)

- **The map:**
  - opens centred on the lead's place;
  - tapping the tracker's place line centres the map on that marker and selects it;
  - pan and centring keep markers clear of the tracker and the place card;
  - marker labels never ellipsize (wrap to two lines).
- **Tracker:** shown after an ending, during the reign.
- **Moments:**
  - goal and achievement moments are blocked while the ending screen is up, and play after it;
  - the queue never drops items: every queued card is shown, including on acts that don't open a conversation (root cause required);
  - the goal moment does not dim the conversation panel;
  - no undimmed flicker between "Goal complete" and "New goal".
- **Ruling outcome moment** from `ruling`.
- **Conversation:** figures stay grounded when the tracker is hidden (after the ending).
- **Room panel:** the last button is never clipped (scroll padding plus a fade cue). Turning-point choices get a distinct treatment and show their warning on the button itself.
- **Fight:**
  - the count sits beside the bar, not on its ticks;
  - at the end both strips stay, or both go;
  - the result card is one plain outcome line plus "What was decided", with the italic preamble removed.
- **Ending screen:**
  - title, tagline, the ≤ 60-word description, the bullets collapsed under "What changed";
  - this walk's achievements as compact badges (not the old marks);
  - the primary button reads `<verb> — <reign_hint>`.
- **Walk list sheet:** "Nara Voss · The Sundial Market · 15 steps · 2h ago".

### W11 fix-1 — what the replay found  *(Oct 4)*

Replayed as Nara to The Bloodless Crown and one ruling. Most of W11 holds. Still wrong:

**Server + data (Codex)**
1. **The reign runs dry after one ruling.** Most petition places are rumoured or gated (`gate_passed`, `stair_open`, `cassian_trust`), so a short route reaches the reign with two petitions in range, one of them the lead's own. New optional location field **`opens_with: string[]`**: when any of these flags is set, the place is `open`, ignoring its rumour and its unlock gate. `visibilityFor` honours it, and so does the client's copy. It goes into `worldVocabulary`. Data: every place that holds a petition gets `"opens_with": ["petitions_open"]`. Audit (must bite): for each ending, from the ending's place with that ending's flags plus `petitions_open`, every petition whose `requires` is met is at a reachable place, and the goal tracker has a tracked petitions goal.
2. **Goal text for one lead.** "Watch the Verdict in the Ring" is shown to Nara, who hears it from the market ("Here"). Use lead-neutral goal text ("Learn how the Verdict ends").
3. **Nara's Ring scene** says "The Sundial Market is open." She has just come from there. Drop that sentence from the two `lead: nara_voss` Ring variants.

**UI (agy)**
1. **The map still opens on the Ring.** `_selectedId ??= _state.currentLocationId` latches the unbound start place on first load. Binding a lead moves them to the market, but the selection, the card and the camera stay on the Ring. When `currentLocationId` changes through bind, rebind or restore, the selection follows it.
2. **Read `opens_with`** in `WorldLocation.visibilityFor`.
3. **Contrast.** The "TURNING POINT · …" caption (`brassHot` on parchment) and the ending's "WHAT CHANGED" / "ACHIEVEMENTS EARNED" captions (`brassDeep` on near-black) are unreadable.
4. **The ending is still long.** Above the fold sit the tagline, the description and the ~100-word reign premise. Collapse the premise into the "What changed" section, so that the title, cost, ≤ 60-word description, badges and buttons fit without scrolling on a 1280×2856 screen.
5. **Clipped head.** After `StageRise.speak` moved to `bottomCenter`, a figure taller than its slot is cut off flat at the slot's top (Arbiter Vell's head, conversation after the ending). Keep the feet grounded and let the figure overflow upward, not clip.
6. **The tracker covers the lead's face** in the room (Nara at the Iron Fane and the market). The room figure's head must clear the tracker.
7. **The first achievement card may not show.** `sand_and_silence` unlocked on the first act of a fresh walk (goal complete + achievement in the same response), yet no card was seen 6.5 s later. On the second such act the sequence played correctly. Reproduce it with a widget test that starts from `_goals == null`. Fix it if it is real.

---

## W12 — Every finding from the nine walks  ·  backend (Codex) + data (cmd) + UI (agy ×2)   *(owner: "FIX EVERYTHING", Oct 4)*

The source is `playtests/2026-10-04-iron-verdict-nine-runs.md`. Every item in it is in scope, P2 polish included. Each item below has one owner. `S` = server (Codex), `D` = data (cmd), `A` = UI lane A (map, rooms, rewind, rebind), `B` = UI lane B (conversation, fights, endings, moments, sheets).

### Decisions taken

- **Every place can be entered.** A place without a scene painting is no longer "travelled past, never entered": the room opens over the **map plate, framed on that place's marker** (the plate paints its places), darkened behind the stage. Choices, drills, petitions and the people who live there all work there exactly as in a painted room. Paintings for the ten sceneless places can come later from the asset lane; nothing waits on them.
- **One source for a fight's strength.** A choice that stages a contested duel carries no `require_traits` (audit fails if it does). What the lead needs is the duel's own rating, sent before the tap, and shown on the button.
- **An ending outranks a death.** When one act both ends the lead's life and meets an ending trigger, the player sees the ending; the death sheet is only for a death that ends nothing.
- **Flagged speech is never left on screen.** Speech streams live, so hygiene that flags a line after the stream sends a revision, and the client replaces the line.

### Server (Codex)

- **S1 · Sceneless places.** Nothing on the server may treat a sceneless place as unenterable: presence, talk, choices, drills and petitions resolve there as anywhere. The wire carries enough for the client to frame the plate on the place (the location's existing map coordinates; confirm they're sent).
- **S2 · Ending beats death.** An act that sets the lead's `dies_with` flag *and* meets an ending trigger returns the ending (as `progression.ending`) with the lead's death recorded as part of it. `lead_dead` / the fall fork is not raised for it. The rebind that follows starts the new lead **in the reign**, with the ending already seen, so it is not shown again.
- **S3 · Rebinding a new lead.**
  - The new lead's `prologue` is returned and is shown (client A).
  - World flags stay: they are history. But goals the previous lead completed are reported as **"Before you"** (`goals.before_you: [{id,text}]`), never as the new lead's done list, and achievements are not re-credited to the new lead for the old lead's deeds.
  - Choices whose `for_leads` was the dead lead are never offered again.
- **S4 · Strength, from the duel.**
  - Each offered choice that stages a contested duel carries `stakes: { opponent, need_strength, have_strength, winnable_now, winnable_at_cap }`, where `need_strength` is the lowest strength at which `contestScore` (with the lead's other traits as they are) meets the opponent's rating.
  - `require_traits` is refused by the loader on such a choice (audit check that bites).
  - Taking it below `need_strength` is allowed and plays the authored loss.
- **S5 · Turning-point hints are derived, not shared.** The "train your arm" sentence is added by the engine only to a contested duel the lead fights. Authored `critical.hint` carries only the story half.
- **S6 · Choice `moves_to`** (new optional field): after the deed the lead stands at that place (it is added to `visited_location_ids` and counts as a move for presence and first meetings). The loader validates it is a real location.
- **S7 · Speech knows who it is talking to and what just happened.** The speech context gains:
  1. the lead's identity: name, role and the authored tie between the speaker and the lead (`ties` on either cast member), so the Duke knows his niece and the Ring's people know their champion. A `first_met` line is used only when no tie exists.
  2. the deeds of this walk that involve the speaker: as `character_id`, as a witness, a ruling on a petition they are party to (with the shown outcome), and the latest deed at this place. Newest first, capped, and filtered by what the player is never sent.
  3. `knows` entries may be gated (`requires`/`forbids`, same predicate as topics); a gated fact is withheld until it is true.
- **S8 · The echo check catches retellings.** Compare each sentence of the reply with the deed's label, echo and summary, after normalising the person (second-person, first-person and the lead's name collapse to one token) and the tense, by content-word overlap. A sentence that restates the deed is dropped. If nothing is left, the reply falls back to the authored line. No word lists: the normalisation is structural (pronoun class from the grammar the model is asked to write in, the lead's name from the cast). Shown on the three lines in the playtest.
- **S9 · Flagged speech is revised on screen.** When post-stream hygiene changes or rejects a reply, a `speech.revise` frame carries the final text (or the fallback), and the stored conversation holds only that. `world_speech.flagged` still logs.
- **S10 · An action in asterisks** is stored and replayed as an action (`kind: 'action'`), not as a question, and is in the Earlier words payload.
- **S11 · Goals.**
  - A goal whose deed is no longer possible is withdrawn: a `by` goal is tracked only when at least one of its `by` choices could still be offered (its `forbids` are not set and its `for_leads` admits the lead).
  - **Next road:** a goal's way gains `via`, the next place on the shortest path along roads the lead can walk now, plus `hops`. When the target is not revealed, the way names only `via`.
  - Done credit is per choice, not per flag: a goal is done only by one of its own `by` choices having been taken.
- **S12 · Rewind.**
  - The response says where the lead was standing at the checkpoint (`view: 'room'` when the checkpoint was taken in a room).
  - Step count comes from the restored checkpoint.
  - Achievement records this walk unlocked after the checkpoint are withdrawn, and `ending_achievements` is recomputed.
- **S13 · `ending_achievements`** lists every achievement this walk's state meets at the ending, each with `new: boolean` (earned now, or already held from another walk). Rewound-away ones never appear.
- **S14 · Latency and logs.**
  - `http.request` logs a real `durationMs` (wall time from request start to response). Show a before and after line.
  - World speech logs one usage line per reply (model, tokens in/out, cost), in the same shape as fight staging.
  - "Fight the Verdict" and "Take the known road" take 4–10 s. Find where the time goes, with the fixed timer. No model call may block `act()` (memory: walkable speech is a chat turn). Report the numbers before and after.
- **S15 · Achievements view:** each achievement carries a `hint` for locked, non-hidden ones (its blurb, phrased as an aim), and a hidden one carries a lead-neutral hint ("One of the walk's endings").
- **S16 · Audits that model the client.** In `audit:iron-verdict`:
  - every choice, drill and petition sits at a place the lead can stand in;
  - every `by` goal can still complete on some route after each of its `forbids` fires;
  - no contested choice has `require_traits`;
  - every `moves_to` is real;
  - every lead can reach every ending that is not `not_for_leads` for them (report which).
  Each check must bite.

### Data (cmd)

- **D1 · Forbids.** `name_the_date`, `keep_the_days_writ`, `read_writ` and `demand_the_paper` forbid `writ_burned`, `writ_sold`, `succession_passed`. Sweep every choice for the same "pointing at something already gone" leak and list what you changed.
- **D2 · Cassian fights the succession.** A `for_leads: cassian_vale` choice "Fight the succession Verdict honestly" at the Ring (requires `writ_burned`, forbids `succession_passed`; sets `succession_fought`, `succession_passed`). It has its own contested duel: Cassian is the challenger with his live traits against Halloran. The win reaches Iron Verdict. The loss is authored and fatal (Cassian dies on the sand: `cassian_dead`, `succession_passed`, never `succession_fought`). The loss meets no ending trigger, so it is a death, and rebinding goes on in the reign.
- **D3 · Remove `require_traits`** from every contested choice (S4).
- **D4 · Turning-point hints:** each `critical.hint` is the story half only, and specific to its choice. "Throw" and "Let Cassian throw" don't mention losing.
- **D5 · Lead-specific text.** Fix each, and sweep for the class:
  - Cassian: "Cassian is not a prisoner…", "Yours is in your coat", "A folded paper with your mark…", and "You are still on the sand" after he has left.
  - Sereth (holding the writ): "the woman under the bronze dial" and "Pick up the mark".
  - Sereth's Verdict Upheld: "The Court owns your calendar".
  - Verdict Upheld on the burn route: "you are paid for it". Use per-lead or per-route variants (`cost_when` keyed on flags if the schema needs it; tell the server task).
  - Broken Ring promises a barricaded Court and a shut Marblegate. Make the reign deliver them (`opens_with`/visibility under `broken_ring`), or cut the promise.
- **D6 · The stair.** `take_the_stair` gets lead-neutral copy that doesn't assume the writ (or requires `writ_read`, then its copy may mention it), and `moves_to` the place the stair comes out at. `nothing_left_unread` requires `writ_read` as well as `stair_taken`.
- **D7 · Goal text:** "Ask Gaunt what his spare key opens" matches the button. Topic: "door on your belt" becomes "key". "Trace the matrix" is replaced with plain words. The Arbiter's "I named it myself" is replaced. "What became of the old rolls?" `forbids: rolls_named`.
- **D8 · Chapter title** names no ending ("Chapter I · The Iron Verdict" or the world's own chapter name, not one of the four endings).
- **D9 · Abbey:** `greywither_abbey` `opens_with: ["abbey_opened"]`. Check every goal's target for "granted but hidden".
- **D10 · Ties:** every cast member who has a relationship with a playable lead names it in `ties` (Duke ↔ Sereth, Gaunt/Hob/Herald ↔ Cassian, and the rest). Gate the Herald's folded-paper fact on the Verdict that leaves it.
- **D11 · Copy:**
  - Kell is "Kell" in the fight narration (no "Dovin" alone).
  - Hob's gate line says what opens him ("Hob Cardew works for the Ring. Win Cassian's trust and he'll talk").
  - The overture's "The Hinge" says how to take a moment again (the clock button).
  - Sereth's opening hall: either Aldric and Barran are present for her start (start flags), or the scene doesn't name them.
  - "Hob's figure is missing when Nara arrives": confirm it's the gate. If so, the room says so.
- **D12 · Deaths:** the death copy is said once (no repeated sentence).

### UI lane A (agy) — `interactive_world_screen.dart`, `map/**`, `world_map_view.dart`, `room/**`, `stage/**`, `world_veil.dart`, `travel_transition.dart`

- **A1 · Enter any place (S1).** The room opens over the map plate, cropped and framed on the place's marker and darkened. The tracker's "Here" always lands in that room.
- **A2 · Room panel.** No springback: the scroll position holds, and a tap lands on the row under the finger. The last row clears the bottom fade, at any row count. Only the tapped row shows a spinner, and the other rows are disabled while it's pending. A reflow during a pending tap can't move the target.
- **A3 · Strength on the button (S4).** A contested choice shows the opponent and "Strength N needed · you have M" before the tap. Below the need it reads as a risk, not a lock.
- **A4 · Echo line:** the line at the top of the room belongs to the deed done **at this place**. After a move, the room shows its arrival line. The deed's echo never follows you.
- **A5 · Map.**
  - "The road is known" only when there's a Travel or known-road button. Otherwise the card names the next road ("2 roads away · via The Rimegate") and draws the route.
  - The tracker shows `via` (S11).
  - Camera clamp: the painting always covers the viewport, at every zoom (fixes the W11 regression).
  - The opening zoom shows the lead's place and its neighbours.
  - Markers under the header scrim or the place card are pushed clear or stay tappable.
  - "World map" works on the first tap.
  - No grey flash: precache the plate before the map shows.
  - The north seam: find out if it's the renderer (tiling, a filter edge) or the painting, and fix the renderer case. Report the painting case.
- **A6 · Rewind:** lands in the room when `view: 'room'`, drops queued moments from after the checkpoint (a held ending card never plays), updates the step count, and shows a spinner on the tapped step only. "FATAL" in red becomes plain words ("Nara falls here").
- **A7 · Rebind:** shows the new lead's prologue, the spinner sits on "Walk as …" not on "Take the fight again", and the "Before you" goals (S3) show in the journal under their own heading.
- **A8 · Death sheet:** no stray small figure of the dead lead.
- **A9 · Ravensgate:** the room is empty and the lead's figure is missing there. Find out why (asset, presence, layout) and fix it.
- **A10 · Training:** a "+1 Strength" moment.
- **A11 · Accessibility:** controls behind opaque sheets are excluded (`BlockSemantics`), and every visible marker has a semantics label.

### UI lane B (agy) — `world_characters.dart`, `world_duel.dart`, `world_ending.dart`, `world_moments.dart`, `goals/**`, `world_identity.dart`, `world_overture.dart`, `world_prologue.dart`, the realm walk list

- **B1 · Conversation.**
  - The conversation tracker's place line goes to the map marker (`onLocationTap`), and the panel stops being hit-testable after "Step away".
  - Enter sends. The first tap on Speak with the keyboard open sends.
  - An *asterisk* action is shown as an action (italic, no "You asked:"), and Earlier words includes actions and is easy to read (S10).
  - `speech.revise` replaces the line in place (S9).
- **B2 · Fights.**
  - The tap control is worded as what it does (the next blow), not "Raise your shield".
  - The result card: the outcome line first, then "What was decided" heading the decision. Every block has its label, and fates come after.
- **B3 · Ending.**
  - Expanding "What changed" keeps the buttons reachable (the sheet scrolls, and the buttons are pinned).
  - "Achievements earned" shows `new` ones as new and held ones as held (S13).
- **B4 · Lead picker:**
  - "Turn them. Stand with one." is replaced with plain words.
  - Each stat has a one-line explanation.
- **B5 · Standing:** the chip and the strip say what standing means in words (no bare "−2 · 1 more").
- **B6 · Petition panel:** it doesn't cover the petitioner. Three rulings fit. A two-line ruling label never overlaps the outcome.
- **B7 · Achievements:**
  - The progress bar reads "3 of 18".
  - Locked achievements are legible and show their `hint` (S15).
- **B8 · Walk list rows** don't wrap mid-phrase ("1d ago" stays whole).
- **B9 · Accessibility** for the sheets lane B owns, as in A11.

### Done when

- `bun run typecheck`, `bun run audit:all`, `flutter analyze lib/features/interactive/` and `flutter test` are all green.
- Every new check bites.
- On the emulator, the orchestrator replays every P0 route with each lead, plus a sample of P1s:
  - the south chain to selling the writ;
  - the north chain to the moot;
  - each sceneless petition ruled;
  - Cassian's throw → Verdict Upheld;
  - Cassian's honest fight (won → Iron Verdict, and lost → death → rebind into the reign);
  - burn → no writ goals.

  Spend cap for the replay: ₹60 / $0.70.
