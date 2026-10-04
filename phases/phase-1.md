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
- **Tracked** is the nearest open goal without `warn`. If every open goal has a `warn`, it is the nearest of those. Ties are broken by file order.
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
