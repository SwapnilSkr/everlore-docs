# Getting the regexes out of the fiction: an independent study

**Status:** study + proposed sequencing, Sept 2 2026. Nothing here is built yet.
**Relationship to [DEVOCABULARY_PLAN.md](DEVOCABULARY_PLAN.md):** independent
re-derivation from the code. It agrees with that plan's diagnosis and its
keep-deterministic calls, and disagrees with its inventory, its cost model, and
— most importantly — its first two phases, which would build a mechanism this
codebase already runs and already pays for.

---

## 0. The one-line finding

`adjudicateSceneEndpoint` is an evidence-carrying, machine-verified,
closed-world presence judge. It runs on **every turn**
(`generation.processor.ts:1251`, inside the unconditional `Promise.all` at
`:1267`). Its answer is consumed for the scene cast **only when the scene
breaks** (`:1882`). On a continuation turn — the majority of turns — that answer
is computed, paid for, and thrown away, and presence is instead decided by the
unverified witness field plus a regex corroboration gate built out of verbs.

> The replacement for the presence word lists is not a new LLM call, a new
> schema, or a new pattern. It is **consuming a judgement we already buy and
> currently discard.**

Everything else in this document is ordered behind that.

---

## 1. Inventory — where my numbers differ

I recounted rather than inheriting. `src` + `worker`, 176 TS files, 41,348 LOC.

| Measure | DEVOCABULARY_PLAN | This study | Why it matters |
|---|---|---|---|
| Uppercase constants | 214 | **298** | Undercount; the split into 66/10/6/4 doesn't reconcile |
| Inline regex operations | not counted | **~298** | A whole second population of `.test(/…/)`, `.match(/…/)`, `.replace(/…/)` |
| Post-prose LLM calls | "three, in parallel" | **five, in parallel** | The replacement budget is computed against this |
| `prose-hygiene.ts` | absent from inventory | **32 regexes** | Not mentioned anywhere in that plan |
| `scene-location-signal.ts` | treated as live | **dead code** | Zero production references |

Constant counts are the wrong unit anyway: `movement-signal.ts` holds 31
constants but they compose into 12 exported functions, of which **3 are
unreachable in production**. What matters is *reachable deciders*, not
alternation strings.

### The rule I'd adopt instead

DEVOCABULARY_PLAN's rule — *"no decision in the narrative pipeline may be made
by matching against a hardcoded list of words"* — is too blunt to be operable.
It immediately needs three carve-outs (safety, closed enums, config), and it
still condemns JSON repair and text normalization, which are load-bearing and
correct. Replace it with one line that needs no exceptions:

> **A regex may verify or normalize. It may not decide what happened in the
> fiction.**

This classifies all ~600 sites with no special cases, and the codebase already
half-obeys it. Compare two functions written by the same hand:

- `hasGroundedWitnessLocationEvidence` (`movement-signal.ts:133`) and
  `hasExactEvidence` (`scene-endpoint-adjudicator.ts:37`) — pure *verification*.
  No vocabulary. Correct, and the model of what to build.
- `hasSceneParticipationGrammar` (`presence-gap-detector.ts:341`) — vocabulary
  *deciding* whether a person was in a room. The thing to remove.

---

## 2. What the pipeline actually is

Verified against `generation.processor.ts`. Corrects the "three parallel calls"
figure the cost argument rests on.

```
pre-stream    buildContextPacket → embed ×2 (+ optional rerank LLM)   ← TTFT
              extractPlayerInteractionSignals   (started, not awaited)
              scoreScene (lexicon, no LLM — deliberately off TTFT)
STREAM        callLLMStreamWithFallback ..............................  TTFT
post-stream   ┌ extractSceneWitness        (620 max tokens)
              │ extractChoiceMetadata      (450)
              ├ adjudicateEntityCandidates                   FIVE calls,
              │ extractCharacterDeaths                       one Promise.all
              └ adjudicateSceneEndpoint    (280)
then          classifyBorderlineIntent (conditional), await interaction signal
async tail    codex deltas → relationship judge → kinship epithets
queues        memory curation, scene/chapter/arc summaries
```

**Every cognitive call in that diagram is the same model.** `AI_MODELS.metadata`
= `gpt-4o-mini` for the witness, the choices, all three judges, the interaction
signals, relationship adjudication and epithet resolution
(`src/ai/models.ts:32-85`). There is exactly one cognitive tier, and the witness
— the most consequential call in the system — carries ~15 fields against a
~2.5K-token static rules prefix at 620 max tokens.

That is the actual root cause, stated more precisely than "conclusions without
evidence": **the hardest semantic job in the product runs on the cheapest tier
with the widest brief, and every time it failed, a word list was bolted on
beside it.** Which means the cheapest possible intervention is not a refactor.

---

## 3. Presence: five deciders, and the one that already works

Presence is not "a witness second-guessed by a regex." It is decided by five
overlapping mechanisms, with `deriveNextSceneState` arbitrating:

| # | Mechanism | Evidence? | Closed-world? |
|---|---|---|---|
| 1 | `extractSceneWitness.present_characters` | no | no |
| 2 | `adjudicateSceneEndpoint.present[{name,evidence}]` | **yes, verified** | **yes** |
| 3 | `adjudicateEntityCandidates` (`not_person` filter) | no | yes |
| 4 | `hasSceneParticipationGrammar` (verb lists) | n/a | n/a |
| 5 | `classifyPresenceCodexGaps` + `isActionableMention` | n/a | n/a |

The regexes metastasize because **they are the arbitration logic between LLM
calls that were never designed to agree.** Every disagreement between two
judges got its tiebreak written in verbs. That is why the lists grow: not one
design mistake repeated 46 times, but a missing authority.

Note that #1 and #2 answer *the same question* under different contracts, and #2
already satisfies every property DEVOCABULARY_PLAN proposes to build:

```37:40:everlore-server/worker/lib/scene-endpoint-adjudicator.ts
function hasExactEvidence(evidence: string, prose: string): boolean {
  const excerpt = comparable(evidence)
  return excerpt.length >= 3 && excerpt.length <= 240 && comparable(prose).includes(excerpt)
}
```

It is temperature 0, it may only return supplied candidates ("this prevents a
second model from inventing a new person", `:42-46`), and every name is dropped
unless its quote is literally in the prose (`:110`).

**So adding citations to the witness would create a third redundant presence
witness rather than retiring one.** The consolidation runs the other way.

And the code already says so, on the break path:

```1875:1885:everlore-server/worker/processors/generation.processor.ts
    // On a boundary the endpoint judge becomes the authoritative cast when it
    // is available. A known old name cannot survive just because the first
    // witness saw it earlier in the prose or an NPC-only cutaway. If the judge
    // is temporarily unavailable, retain the existing witness-only fallback so
    // an auxiliary model outage never erases a valid live scene.
    const candidates = sceneBroke
      ? [
          ...(endpointAdjudication.available ? endpointPresenceNames : (parsed.present_characters || [])),
          ...partyNames,
        ]
      : [...priorPresent, ...(parsed.present_characters || [])]
```

On `sceneBroke` the verified judge wins and the witness is merely its outage
fallback. On `!sceneBroke` the judge's answer is discarded and we fall back to
the unverified witness — which is then gated by the verb lists in
`deriveNextSceneState:244`. **The regex presence gate is only load-bearing on
continuation turns, because on scene breaks an evidence-carrying judge already
replaced it.** The pattern is proven in production; it is just not wired up on
the majority path.

---

## 4. Three things that change the plan

### 4.1 Time is inverted from its own documentation

`time-skip-signal.ts:10-11` states: *"Used only as a fallback: a model-reported
`time_elapsed` always wins."* That is false in the live code:

```1578:1581:everlore-server/worker/processors/generation.processor.ts
  const witnessTimeLabel = !isContinuation ? parsed.time_elapsed || undefined : undefined
  const playerTimeLabel = !isContinuation ? detectNarratedTimeSkip(parsedPlayerInput.raw) || undefined : undefined
  const narratedTimeLabel = !isContinuation ? playerTimeLabel : undefined
  const effectiveTimeAdvance = timeAdvanceLabel || actionTimeAdvanceLabel || narratedTimeLabel
```

`narratedTimeLabel` is assigned `playerTimeLabel` only. `witnessTimeLabel` is
computed, never consulted for calendar advance, and used solely to populate
`detected:` in the signal ledger (`:3397`). The calendar is **100% regex** and
the model's answer is discarded.

Consequence: "retire the 4 time constants" is not a cleanup, it is **granting a
model authority it has never held over an irreversible quantity** (story dates
mutate the timeline). It belongs last, not in the middle. The stale comment
should be corrected immediately regardless — it will mislead the next reader
into assuming a safety property that does not hold.

### 4.2 Party is a privilege inversion

Party is the least-evidenced signal in the system: pure regex, **no witness
field at all**, and it *bypasses* the corroboration gate that every other
arrival must pass.

```244:244:everlore-server/src/services/scene-state.service.ts
    const justified = openingScene || travelKeys.has(k) || corroborated.has(k)
```

Plus a force-add at `:269-275` even when the witness omitted them, and exemption
from the stale-cast purge at `generation.processor.ts:1898`. So the weakest
signal carries the strongest privilege — which is exactly why the
"I take Halvard by the collar" bug propagated an assaulted steward through every
later scene. This is worth fixing *independent of* the de-regex project, and it
is the one area where DEVOCABULARY_PLAN's "add a new evidence-carrying field" is
exactly right, because there is no existing judge to consolidate with.

### 4.3 The harness is largely built, and the test suite does not exist

DEVOCABULARY_PLAN's Phase 0 ("replay corpus + shadow-compare, prerequisite for
everything") is mostly already there:

- **`signal_ledger`** (`src/models/signal-ledger.model.ts`) already records
  per-turn `detected` vs `committed` for movement/time/party/kinship/presence,
  plus `player_corrected` as a precision ground truth and `miss_candidates` as
  a recall ground truth. Its own header says it exists because *"you cannot tune
  that trade by vibes."* `scripts/signal-ledger-report.ts` aggregates it.
- **`nsfw-extraction-probe.ts`** is already *"a no-write replay of the real
  post-narration extraction fan-out"* calling production extractors with an
  `onRaw` hook.
- **`player_input` and `ai_response` are persisted per turn**
  (`world-event.model.ts:111-180`), which is sufficient input to re-derive.

Missing is small and specific: raw witness/judge JSON is never stored (the
`onRaw` hook exists and nothing subscribes it), and there is no batch runner
reading events from Mongo. That is a task, not a phase.

What is genuinely missing is much more alarming: **there are zero test files and
no test runner.** No vitest/jest/mocha, no `*.test.ts`, no CI workflow. There
are ~32 `bun` audit scripts, many pure-function with real expected-vs-actual
assertions — a good substrate — but nothing runs them together and nothing runs
them automatically. Before changing the semantic core of the product, that is
the real gap, and `audit:movement` currently spends part of its time asserting
against three functions nothing calls.

---

## 5. Proposed sequencing

Ordered by value ÷ risk, not by the order the lists appear in the code.

### Phase 0 — subtraction only, zero behaviour change

Hours, not weeks. Nothing here can regress the fiction.

- Delete `worker/lib/scene-location-signal.ts` — whole file, zero production
  references (`establishesSceneLocation` is its only export).
- Delete `isGroundedPlayerDestination`, `refinePhysicalDestination`,
  `isExplicitPlayerLocationChange` from `movement-signal.ts` and the constants
  only they use. (`resolvePossessiveRoomName` **stays** — live internally at
  `:540`.) Drop their cases from `audit:movement`.
- Fix the false `time-skip-signal.ts` doc comment (§4.1).
- Wire the existing audit scripts into one runner + CI. This is the safety net
  every later phase depends on.

### Phase 1 — consume the judge we already pay for  ← *start here*

Feed `adjudicateSceneEndpoint` into the cast on continuation turns, not only on
scene breaks. Its brief — *who is physically co-located with the player at the
final moment* — is already the correct question for a continuation.

- **Shadow first:** compute the endpoint cast on continuations, ship the current
  answer, log disagreement into `signal_ledger` (which already has the shape).
- **Then flip**, and `hasSceneParticipationGrammar` loses its job by
  construction: the corroboration gate becomes "the judge cited verified prose."
- Retires the whole `tierFor` ladder — `SPEECH_VERBS`, `ACTION_VERBS`,
  `PERSON_POSSESSIONS`, `POSSESSED_THING_ACTS_VERBS`, `TITLE_WORDS`.
- **Cost: zero new LLM calls, nothing new pre-stream.** Possibly a modest
  `maxTokens` lift on the judge (280 today) since continuations carry more
  names. This is cheaper than DEVOCABULARY_PLAN's Phase 1+2 *and* it deletes
  more, because it removes a decider instead of adding evidence to one.

### Phase 2 — measure the tier before refactoring anything else

Cheap, and it determines whether later phases are needed at all. Everything
cognitive is `gpt-4o-mini`; the witness carries ~15 fields at 620 max tokens.
On the Phase 1.5 corpus, compare: (a) today, (b) witness split further into
narrower briefs, (c) one tier up on the witness only.

If a narrower brief or one tier fixes the failure modes the word lists patch,
Phases 3–5 shrink or disappear — and that is a far better outcome than building
schema for all of them. DEVOCABULARY_PLAN files this as "Open question 1,
measure don't guess"; I agree entirely and would therefore **gate the remaining
work on it** rather than leave it open.

### Phase 1.5 — finish the harness (small, parallel with the above)

Subscribe the existing `onRaw` hooks and persist raw witness/judge JSON; extend
`nsfw-extraction-probe.ts` from an inline fixture to a Mongo-backed batch runner
over stored `player_input` + `ai_response`. Report through
`signal-ledger-report.ts`. Note that re-extraction is non-deterministic, so this
measures distributions, not exact replay — fix a seed corpus and run n>1.

### Phase 3 — movement and place

The messiest area, because `narratedArrival` requires a *player-input regex* and
explicitly ignores the witness's `viewpoint_moved` (`:1473-1489`). Order:

1. Have the witness cite movement the way it already cites location — the
   `location_evidence` + `hasGroundedWitnessLocationEvidence` pair already
   exists and already caught a fabricated citation.
2. Let the witness treat the player turn as a first-class movement source. It
   already receives `playerInput` and `location_evidence_source: 'player'`
   already exists — that is precisely what `detectNarratedMovement` compensates
   for.
3. Only then drop the regex requirement from `narratedArrival`.

Place *admissibility* keeps a **structural** residue rather than noun lists:
a label identical to a pronoun, or article-led and generic, is not a name. Model
it on `readsAsBareTitle` / `readsAsCommonNoun`, which judge text *shape* and
generalise across language and genre.

### Phase 4 — party

Fix the privilege inversion (§4.2) *first* — make companionship earn its place
like every other arrival — then add the evidence-carrying
`party: [{ name, evidence, joined | parted }]` field. New schema is justified
here because nothing existing answers this question.

### Phase 5 — time, last

A behaviour change, not a cleanup (§4.1). Highest blast radius per unit of
benefit: a false advance mutates story dates. Its own gate, its own corpus
slice.

### Keep deterministic — permanently

I agree with DEVOCABULARY_PLAN on all of these and would add two categories it
does not cover:

- **`input-guard.ts` safety lists.** Correct as-is. Asking a model whether text
  is safe to send to a model is circular, and a safety gate must not depend on
  API availability or latency. I would add: nor on *cost*, and it must stay
  offline-auditable.
- **`nsfw-classifier` fallback lists.** Already has an LLM path; the lists are
  the Mongo-backed offline fallback.
- **Closed enums** (`VALID_SCENE_TAGS`, `RELATION_KIND_SET`, the `movement`
  enum, `MOVEMENT_VALUES`). These are not heuristics — they are what makes a
  model's answer checkable. Deleting them removes the verification, not the
  vocabulary.
- **JSON repair and prose hygiene** (`structured-output.ts` ~38 regexes,
  `prose-hygiene.ts` ~32) — *not in DEVOCABULARY_PLAN's inventory at all*. These
  operate on text shape, never on fiction semantics. Legitimate under §1's rule.
- **Normalization** (`normalizeEntityName`, `comparable`, `clean`). Legitimate
  and necessary: this is the machinery that makes evidence verification possible
  in the first place.

---

## 6. Design principles for the harness

What "a proper harness" means here, stated so it can be checked:

1. **One authority per question.** Every narrative fact has exactly one decider.
   No arbitration-by-regex between disagreeing models.
2. **Closed-world subcalls.** A judge selects from a supplied candidate set and
   cannot invent. Already true of both existing judges.
3. **Evidence-carrying and machine-verified.** Every claim quotes the prose; the
   server verifies literal containment. Already true in three places.
4. **Deterministic code verifies; models decide.** The §1 rule.
5. **Fail direction declared per field.** Fail *closed* on admission (do not add
   a person or place), *open* on continuity (do not remove someone established).
   This is the bias already chosen implicitly; make it explicit per field.
6. **Nothing new before the stream.** TTFT is untouched by construction in every
   phase above.

---

## 7. Open questions I am not guessing at

1. **Does the endpoint judge degrade on continuations?** Its prompt is written
   for an endpoint ("the FINAL MOMENT"). On a long continuation the final moment
   *is* the current cast, so I expect it to hold — but Phase 1 ships in shadow
   precisely because I do not want to assume it.
2. **Do the structural tests assume English capitalisation?** `readsAsCommonNoun`
   and `appearsMidSentence` rely on case. That breaks for non-cased scripts and
   for German nouns. Worth deciding before Phase 3 hardens the pattern. (Same
   question DEVOCABULARY_PLAN raises; I have no better answer yet.)
3. **What is the real per-turn cost today?** No dollar figures exist anywhere in
   the code — only per-call `maxTokens` and `tokens_in`/`tokens_out` on
   `generation_logs`. Any "≤ +15% cost" gate is unmeasurable until that is
   derived from the logs. Build the measurement before setting the budget.

---

## 8. Live state

Unchanged from DEVOCABULARY_PLAN §6: server `729ab07` is in prod, app 1.0.4
(vc7) is on the Play alpha (closed) track.

Nothing proposed here requires reverting either. Phase 0 is pure subtraction of
unreachable code; Phase 1 ships in shadow mode and changes no behaviour until
its disagreement log is understood. The widened lists in prod are better than
what preceded them and wrong by design — both remain true, and neither is
urgent enough to justify pulling a release.
