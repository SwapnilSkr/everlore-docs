# Scene State — the authoritative present moment

**Status:** built and live-verified 2 Sep 2026 (uncommitted, `everlore-server`).
**Audit:** `bun run audit:scene-state` (30 cases). **Repair:** `bun run repair:split-identities`.

---

## Why this exists

A player grabbed a character by the collar, released him, and four turns later
the character said *"Now. Let go of me."* A few turns after that, the King
answered a line the player had spoken **before the King entered the room**. Two
characters left behind in a dining hall stood in a different room for seven
turns. A character who walked out came back through the door repeating a line
he had already delivered.

These looked like four bugs. They were one absence.

Everything in the world model answered a question about the **past** — the event
ledger is what happened, the codex is who someone is, memories are what is worth
recalling, the location graph is what places exist. Nothing owned the only
question the narrator must answer correctly every single turn:

> Right now, in this room, what is true?

Without an owner, that answer was re-derived each turn from whatever survived
the 6-turn recent-event window. Three structural consequences followed.

### 1. State was reconstructed from history, not maintained

- `mutable_state` accreted into a 12-slot FIFO with no ordering and no
  supersession. One live card held *"Has left the council room"* and *"Is being
  invoked by Aldric"* simultaneously.
- The recent-turn ledger read each event's frozen `codex_deltas[].mutable_state`
  and labelled every entry **"current state"** — so turn 19's *"pinned against
  the wall"* and turn 22's *"furious in the doorway"* arrived with equal
  authority.
- Retirements were applied to the **card**, never subtracted from the stored
  **events**, so a retired fact kept re-entering the prompt for the rest of the
  window.

A fact's life ended when it fell off a FIFO. That is a memory-management
artifact, not a story event.

### 2. The loop was open in both directions — and the information was inverted

- The **narrator was never told who is in the room.** `present_characters` was
  computed, stored, shipped to the app's presence UI, and deliberately withheld
  from the prompt (`prompt-builder.ts`: *"present_characters is not otherwise
  prompt-injected"*).
- The **extractor was told** (`metadata-extractor.ts`: *"the people present at
  the end of last turn are listed in CONTEXT below"*).

The component that *wrote* the physics didn't know the roster; the component
that *read* the prose did. And the corroboration gate applied **only to unknown
names** — a character with a codex card was admitted unconditionally, which is
precisely how two carded characters teleported into the council room.

### 3. State writes were advisory, not transactional

Turn 21's projection threw `E11000` on the `entities` unique index, wrote
`status: pending`, **logged no anomaly**, and the story kept generating on state
everyone believed was current. The repair sweeper retried forever against a
deterministic collision. Root cause: the character's identity was split across
two entity rows (a witness stub `cedric` and the card's row still named
`crown prince cedric`), so the converging rename could never land.

---

## The model

Stored on the **event** (`data.scene_state`), not in a mutable per-instance
document — so it is a rebuildable projection like every other derived state
here, and rewind is free: delete the turns and the previous snapshot is current
again.

```ts
SceneStateDoc {
  as_of_sequence: number
  place: { entity_id, name } | null
  cast: Array<{ entity_id, name, since_sequence, source }>
  physical: Array<{ kind, statement, actors[], since_sequence }>
  scene_broke: boolean
}
```

`source` is provenance: `opening | arrival | travel_party | carried`. A cast
entry nobody can justify is a hallucination, not a character.

`kind` is `restraint | contact | posture | held`.

## The three invariants

**1. The cast is a CLOSED set changed only by justified transitions.**
Carrying someone forward is free (a quiet character must not flicker out).
Admitting someone new requires justification: a confirmed travelling companion,
or the turn's prose independently naming them. **This bar applies to carded and
uncarded people identically** — that symmetry is the phantom-presence fix.

**2. The narrator reads it; the extractor diffs against it.**
A hard `SCENE STATE` prompt section states the room positively — who is here,
since when, and what physical configuration holds — so a second entrance is a
*contradiction* rather than a plausible next sentence. The extractor's job is a
diff (who arrived, who left, what opened, what closed), which can be validated;
never a re-derivation, which invites hallucination.

**3. Every physical fact is opened AND closed by a story event.**
A grip ends when a release is narrated, when an actor leaves, or on a scene
break — never by falling off a cap. A new posture or restraint supersedes that
person's previous posture (you cannot be seated and pinned to a wall at once).

## Contradictions are rejected, not folded in

`scene_contradiction` anomalies: `arrival_of_present`, `departure_of_absent`,
`physical_actor_absent`, `uncorroborated_arrival`. Plus `projection_failed`,
`identity_collision`, `stale_scene_state`.

---

## Hard-won details (each cost a live playthrough to find)

- **Identity keys must be title-insensitive.** The extractor returns the
  canonical *"Crown Prince Doran"* on one turn and the prose's bare *"Doran"* on
  the next. Plain normalization put the same man in the room twice, and neither
  copy could be departed by naming the other. `sceneIdentityKey()` strips
  honorifics; a purely titular label (*"the King"*) keeps its full form.
- **The corroboration set must use the SAME key.** Keyed on full names while
  scene state looked up stripped keys, every titled character was refused entry
  to their own scene.
- **The player is an actor but never a cast member.** The witness correctly
  reported `actors: ["Aurelian Marek", "Crown Prince Doran"]` for a collar grab,
  and the fact was discarded every time because the player — deliberately never
  listed in their own scene — failed the "actors must be present" check. Pass
  `protagonistNames`.
- **Match the player's name by token.** The persona is *"Aurelian Marek"* while
  the prose says *"Aurelian"*, so whole-name comparison never matched and the
  player was listed in their own cast.
- **A close needs cited, machine-checked evidence.** Asked *"which of these
  ended?"*, a small model echoes the whole list back — it closed a collar grip
  on the very turn the player wrote *"I hold him tighter"*. `physical_state_closed`
  is `{statement, evidence}` and the excerpt is verified verbatim against the
  prose + player turn; an uncitable close is discarded and the grip stays on.
  Same discipline the codebase already applies to location and bond evidence.
- **`'processing'` is healthy, not stale.** Projections run async on the turn
  tail, so the previous one is usually still in flight when the next turn
  starts. The fence WAITS for it; treating it as staleness fires an error every
  healthy turn and drowns the real signal.
- **Bare honorifics are not people.** *"Prince"* on its own is how the prose
  refers to someone already present; admitting it mints a second, nameless copy.

## Migration

`loadCurrentSceneState` bootstraps a session that predates scene state from the
last turn's `present_characters` — but filtered to those the last two turns'
prose actually names. Seeding it verbatim would preserve existing phantoms
forever, since carry-forward never questions an established cast member. A
genuinely quiet character drops for one turn and returns when the story mentions
them. Verified on the live broken instance: `[Isolde, Lyra, Aldric, Cedric]` →
`[Aldric]`.

## Live verification (2 Sep 2026)

Reproduced on a fresh world through the real WebSocket, and on the actual
affected playthrough through the Android app:

| behaviour | result |
|---|---|
| grip opens on grab | ✅ |
| grip **persists** through *"I hold him tighter"* | ✅ (was: closed) |
| grip closes on *"I release Doran"* | ✅ (was: still gripped 3 turns later) |
| no re-grip on later turns | ✅ |
| leaving a room drops the old room's cast | ✅ |
| a returning character's arrival is narrated once, not twice | ✅ |
| uncorroborated carded characters refused | ✅ — on the live instance the model
  still reported Isolde and Lyra present at turn 32; both were rejected and
  logged |

## Files

`src/models/scene-state.model.ts` · `src/services/scene-state.service.ts` ·
`src/services/context-packet.service.ts` (packet field) ·
`src/utils/prompt-builder.ts` (SCENE STATE section + ledger supersession) ·
`worker/lib/metadata-extractor.ts` (physical-state extraction) ·
`worker/lib/structured-output.ts` (sanitizers) ·
`worker/processors/generation.processor.ts` (derivation, corroboration, fence) ·
`worker/processors/character-projection.processor.ts` (ordering, poison pill) ·
`src/services/entity-graph.service.ts` (`absorbEntityInto`, collision-safe rename) ·
`src/utils/record-anomaly.ts`
