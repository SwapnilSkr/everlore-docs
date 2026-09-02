# Replacing the word lists: a plan

**Status:** proposed, Sept 2 2026. Nothing here is built yet.
**Rule this serves:** no decision in the narrative pipeline may be made by
matching against a hardcoded list of words.

---

## 1. What is actually there

214 uppercase constants across `worker/lib`, `src/utils`, `src/services`. Of
those, **66 are vocabulary-driven** — they decide something by asking whether a
token appears in a hand-written alternation. They fall into four groups, and
only the first is in scope.

### A. Narrative-truth deciders — **REPLACE** (~46)

These answer *"what happened in the fiction this turn?"* — a semantic question
being decided by a spelling test.

| File | Constants | What it decides | Why it exists |
|---|---|---|---|
| `presence-gap-detector.ts` | `ACTION_VERBS`, `SPEECH_VERBS`, `PERSON_POSSESSIONS`, `POSSESSED_THING_ACTS_VERBS`, `TITLE_WORDS` | Is this named person physically in the scene? (`confirmed` tier) | The witness pass reported people who were never in the room. Rather than make the witness accountable, a second opinion was built out of verbs. |
| `movement-signal.ts` | `DIRECTED_VERB`, `DEPARTURE_VERB`, `DIRECTION`, `ESCORTED`, `EXPLICIT_DESTINATION`, `EXPLICIT_LODGING_DESTINATION`, `ABSTRACT_DESTINATIONS`, `DESTINATION_STOP_WORDS`, `GENERIC_LOCATION_TOKENS`, `LOCATION_ACTION_WORDS`, `LOCATION_CLAUSE_WORDS`, `LOCATION_PRONOUNS`, `PLAYER_GROUNDED_GENERIC_DESTINATIONS` | Did the viewpoint move? Is this label a real place? | The witness under-flagged moves (cursor stuck) and over-named places (phantom travel). Both were patched with vocabulary instead of evidence. |
| `party-signal.ts` | `JOIN_SUBJECT`, `JOIN_PLAYER`, `JOIN_PLAYER_BARE`, `JOIN_IMPERATIVE`, `JOIN_TOGETHER`, `PART_SUBJECT`, `PART_DEPART`, `SOLO_TRAVEL`, `SOLO_VERB`, `WITH_YOU` | Who is travelling with the player? | No witness field for companionship at all, so it was inferred from the player's phrasing. |
| `time-skip-signal.ts` | `AMOUNT`, `UNIT`, `DURATION_PASSAGE`, `SPEND_DURATION` | How much story time passed? | The witness's `time_elapsed` was unreliable; a parser was added beside it. |
| `entity-graph.service.ts` | `GENERIC_PLACE_NOUNS`, `LOCATION_TOKEN_STOP`, `POSSESSIVE_VAGUE_ROOM`, `AREA_KINDS` | Is this place label specific enough to be a graph node? | Junk places ("the quiet road") were entering the graph. |
| kinship + identity | `POSSESSIVE_KIN`, `METAPHOR_PATTERNS`, `CUES` (×2), `PLAYER_KEYS`, `PLAYER_ALIASES`, `ABSTRACT_NON_PERSON_TERMS`, `NON_PERSON_ROLE_LABELS`, `DETERMINER_PREFIX` | Is this a kin claim? Is this token a person at all? Is it the player? | Kin ties and person-hood were being inferred from prose shape without the model ever being asked. |
| `player-input-parser.ts` | `CORRECTION_CUES`, `REPORTED_SPEECH` | Is the player correcting canon, or speaking in-character? | Player intent was never asked for; it was pattern-matched. |

**The common cause is one design decision, not 46 mistakes.** Every one of these
exists because *a model was asked for a conclusion but never for its evidence*,
so the conclusion could not be checked — and an unverifiable conclusion gets
second-guessed by a regex. The lists are the scar tissue of that gap.

### B. Safety gates — **KEEP deterministic** (10)

`input-guard.ts` (`MINOR_TERMS`, `MINOR_AGE_PATTERNS`, `SEXUAL_TERMS`,
`VISUAL_SEXUAL_TERMS`, `KIN_TERMS`, `KIN_ADDRESS_TERMS`,
`POSSESSED_KIN_ADDRESS`) and `nsfw-classifier.ts` (`FALLBACK_WORDS`,
`EXPLICIT_PATTERNS`, `AMBIGUOUS_PATTERNS`).

These must not become model calls:
- The input guard exists so certain content **never reaches a model at all**.
  Routing it through a model to make that decision is circular.
- A safety gate cannot depend on API availability or latency.
- `nsfw-classifier` already has an LLM path; the list is its offline fallback and
  is Mongo-backed (`nsfw_lexicon`) rather than hardcoded.

The no-word-list rule is about *narrative truth*, not *safety*. Applying it here
would be a regression.

### C. Closed enums validating model output — **KEEP** (6)

`VALID_SCENE_TAGS`, `MOVEMENT_VALUES`, `RELATION_KIND_SET`, `STUB_STATUSES`,
`SCHEMA`, `SIGNAL_SCHEMA`. These are not heuristics; they are the contract the
model's output is validated against. Keeping them is what makes a model's answer
checkable at all.

### D. Infrastructure — **KEEP** (4)

`BLOCKED_UPDATE_KEYS`, `MAX_GUIDE_FLOWS`, `STUB_SOURCE_EVENTS_MAX`,
`MIN_SALVAGED_EVIDENCE`. Config and limits, no vocabulary semantics.

---

## 2. The replacement: evidence, not vocabulary

Two patterns already exist in this codebase and both work. Neither is new.

**Pattern 1 — machine-checked witness citation.** `location_evidence` +
`hasGroundedWitnessLocationEvidence`: the model must quote the exact sentence
that proves its claim, and the server verifies that quote literally appears in
the prose. Already caught a fabricated citation in a live run (seq 5, run D:
witness cited a sentence from the narrative while sourcing it to the player).
`physical_state_closed[].evidence` uses the same discipline and caught the model
echoing back closes that never happened.

**Pattern 2 — structural test.** `readsAsBareTitle(token, prose)` decides "is
this a title?" by checking whether the story lowercases the word on its own
standalone occurrences. Zero vocabulary. Generalises to any language or genre.

**The thesis:** make every narrative claim *evidence-carrying*, then verify the
evidence structurally. A claim the model cannot cite is dropped. This replaces
"do these verbs appear?" with "show me the sentence, and I will check it is
real." It is strictly stronger: it cannot be defeated by an unusual verb, and it
cannot be satisfied by a hallucination.

### Why this does not cost more

The critical fact: **the post-prose tail already runs three LLM calls in
parallel**, and the cheap-model pass that would carry the evidence already runs
every turn.

Turn timeline today (`generation.processor.ts`):

```
pre-stream   buildContextPacket (DB + embeddings)          ← TTFT-critical
             extractPlayerInteractionSignals (parallel)
STREAM       callLLMStreamWithFallback  ......................  TTFT
post-stream  adjudicateEntityCandidates ─┐
             extractSceneMetadata        ├─ already parallel, off TTFT
             adjudicateSceneEndpoint    ─┘
later        extractCharacterCodexDeltas, memories, summaries
```

`extractSceneMetadata` **already returns** `present_characters`,
`characters_departed`, `current_location`, `player_destination`,
`viewpoint_moved`, `movement`, `time_elapsed`, `physical_state_opened/closed`.
The word lists exist to second-guess exactly these fields.

So the replacement is **not more calls**. It is *the same call, asked for its
reasons*:

- Adds output tokens (one short citation per claim), not a round trip.
- Costs nothing on TTFT — the prose has already streamed.
- Estimated delta: ~150–400 output tokens/turn on an already-cheap model.
- Where a genuinely separate judgement is needed (party membership, player
  intent), it joins one of the **existing** parallel post-stream calls rather
  than adding a fourth.

**Latency budget rule for this work:** nothing new before the stream. The
pre-stream path stays exactly as it is.

---

## 3. Phases

Each phase is independently shippable and independently revertible.

### Phase 0 — Replay corpus and shadow harness *(prerequisite for everything)*

You cannot stress-test this without a fixed corpus; live playthroughs are
non-deterministic and each one costs money.

1. Capture **every turn we generate from now on** — prose, player input, witness
   output, resolved decisions — into a replay corpus (the event ledger already
   holds most of it; add the raw witness JSON).
2. Backfill the corpus from existing saves (33 instances on the dev DB).
3. Build `shadow-compare`: replay a corpus turn through **both** the old
   vocabulary gate and the new evidence gate, record every disagreement.
4. Target corpus: **≥500 turns** spanning fantasy / modern / sci-fi worlds, GM +
   sentient + character archetypes.

Nothing changes behaviour in this phase.

### Phase 1 — Evidence-carrying witness

Extend the `extractSceneMetadata` schema so every claim carries a citation:

```
present_characters: [{ name, evidence }]        // the sentence showing them act
characters_departed: [{ name, evidence }]
current_location:   { name, evidence, source }  // exists today
party:              [{ name, evidence, joined|parted }]   // NEW field
time_elapsed:       { amount, unit, evidence }
```

Server-side verification is **structural**, no vocabulary:
- the citation must literally appear in the prose (existing helper), and
- the citation must contain the claimed name (or a surface of its card), and
- for a departure/join, the citation must contain the player's or character's
  name — not merely the place.

Run in **shadow mode**: compute both, ship the old answer, log disagreements.

### Phase 2 — Retire the presence lists

Once shadow disagreement is understood, `hasSceneParticipationGrammar` is
replaced by "the witness cited a verified sentence for this person." Deletes
`ACTION_VERBS`, `SPEECH_VERBS`, `PERSON_POSSESSIONS`,
`POSSESSED_THING_ACTS_VERBS`.

**This is the one that already burned us:** a brother who "gave a slow nod" was
ruled absent from his own scene for three turns while speaking in it.

### Phase 3 — Retire movement and place lists

`detectNarratedMovement` and `isSafeWitnessLocationCandidate` become:
- movement = the witness cited a verified sentence showing the viewpoint moving;
- a place label is admissible if it is **structurally** a place — the witness
  names it as the setting AND cites the establishing sentence. Junk labels
  ("the quiet road", "the last report's location") fail because no sentence
  establishes them as the current setting, not because a noun is missing from a
  list.

Deletes 13 constants in `movement-signal.ts` and 4 in `entity-graph.service.ts`.

### Phase 4 — Retire party and time lists

Party membership becomes a witness field with citations (it has never had one —
that is why "Neva, walk with me" was invisible and "I take Halvard by the
collar" enrolled him). Time skip likewise. Deletes 10 + 4 constants.

### Phase 5 — Kinship, identity and player intent

`POSSESSIVE_KIN`, `METAPHOR_PATTERNS`, `CUES`, `ABSTRACT_NON_PERSON_TERMS`,
`NON_PERSON_ROLE_LABELS`, `CORRECTION_CUES`, `REPORTED_SPEECH`. These fold into
the codex/kinship extraction pass, which is already an LLM call.

---

## 4. Stress testing — the gate for going live

Nothing ships to prod until all of this passes.

**Corpus replay (deterministic, free after capture)**
- ≥500 turns replayed through old and new.
- Every disagreement classified: new-correct / old-correct / both-wrong.
- Ship only when new-correct ≫ old-correct **and** no new-wrong class is
  systematic.

**Adversarial cases** — each currently a live or historical bug:
- a character who acts with an unusual verb (the `gave`/`lets` case);
- a person named but elsewhere ("the rations Bram had noted", "Mara went to the
  capital");
- a place mentioned but not visited (phantom travel);
- plural and multi-word places ("root cellars", "the north wall");
- grabbing someone (must not enrol them as a companion);
- "walk with me" / "ride back alone";
- a character addressed by kin term, not name ("Brother, walk with me");
- non-English / invented place and person names.

**Live playthroughs**: ≥5 worlds × ≥20 turns across all three archetypes and at
least three genres, with every projection audited — the loop already used this
session, but broadened past one fantasy keep.

**Cost + latency gates**
- TTFT: **no regression**, measured p50/p95 (nothing added pre-stream).
- Post-prose tail: p95 not worse than today.
- Cost/turn: ≤ +15% on the cheap model; alert if exceeded.

**Regression suite**: every audit green, plus new audits per retired list
asserting the *behaviour* now holds without the vocabulary.

---

## 5. Open questions

1. **Model choice for the witness.** Evidence-carrying output demands more of it
   than today's field-filling. `MODEL_METADATA` may need to move up a tier; that
   is a real cost decision and should be measured on the corpus, not guessed.
2. **What happens when the witness cannot cite?** Proposal: fail *closed* for
   admission (do not add the person/place) and *open* for continuity (do not
   remove someone already established). Continuity errors are cheaper than
   phantom errors — this is the bias we already chose, now made explicit.
3. **Non-English prose.** Structural tests must not assume English
   capitalisation. Worth deciding before Phase 2 hardens the pattern.

---

## 6. Current live state (as of writing)

Not part of the plan, but the plan assumes we know where we stand:

- Server `729ab07` **is deployed to prod** — includes the widened lists.
- App **1.0.4 (vc7) is on the Play alpha track** (closed testing, not public).

If "not going live" means these should be pulled back, that is a separate
decision to make explicitly. The lists as they stand are *better* than they were
this morning (three real bugs fixed) and *wrong by design* (this document).
