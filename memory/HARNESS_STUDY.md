# Getting the regexes out of the fiction: an independent study

**Status:** study + proposed sequencing, Sept 2 2026. Nothing here is built yet.
**Relationship to [DEVOCABULARY_PLAN.md](DEVOCABULARY_PLAN.md):** independent
re-derivation from the code. It agrees with that plan's diagnosis and its
keep-deterministic calls, and disagrees with its inventory, its cost model, and
— most importantly — its first two phases, which would build a mechanism this
codebase already runs and already pays for.

---

## 0. The one-line finding

**Revised Sept 2 after review — see [§9](#9-revisions-after-review). Three
claims below were wrong or imprecise; the sequencing survived, the framing did
not.**

`adjudicateSceneEndpoint` is a closed-world presence judge whose every claim
carries a prose citation. It runs on **every turn**
(`generation.processor.ts:1251`, inside the unconditional `Promise.all` at
`:1267`), and two of its four fields are already consumed on every turn — the
scene-break decision at `:1618` is partly its `playerViewpointAtEnd &&
sceneTransition`. But its `present[]` — the cast itself — is consumed **only
when the scene breaks** (`:1882`). On a continuation turn that array is
computed, paid for, and discarded, and presence is instead decided by the
unverified witness field plus a regex corroboration gate built out of verbs.

> The replacement for the presence word lists is not a new LLM call, a new
> schema, or a new pattern. It is **consuming one array of a judgement we
> already buy, already trust with the scene-break decision, and already discard
> on the majority path.**

That the judge has been trusted with scene-break on continuations, under this
exact prompt and this exact input, for as long as scene state has shipped is the
best available evidence that it does not degrade there.

**It is not a superset swap.** The gate it would replace tests a property the
judge's verifier does not (§3.1). That has to be closed first, not inside
Phase 1.

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

### 3.1 But the verifier checks fabrication, not relevance

`hasExactEvidence` runs `.includes()` over the **whole passage**. The prompt
demands proof of presence *at the final moment*; the verifier accepts a quote
from paragraph one. So the property actually guaranteed is "this excerpt is not
invented," not "this excerpt proves presence at the endpoint." Machine-checked
against invention; **model-trusted on relevance.**

And the hole is precisely the bug the gate we want to delete was written for:

```1972:1977:everlore-server/worker/processors/generation.processor.ts
    // Being NAMED is not being PRESENT. An earlier version accepted any
    // occurrence of the name, and a passing reference to something a character
    // had said a day's ride away ("the rations Bram had noted") put him at the
    // top of a ruined watchtower, where carry-forward kept him for the rest of
    // the scene. Require the prose to show the person actually participating —
    // speaking, acting, being addressed — using the same evidence patterns the
```

`hasSceneParticipationGrammar` tests **person-specific grammar**;
`hasExactEvidence` tests **literal containment**. Different properties. Swapping
one for the other drops the check that catches remote-mention, into a
carry-forward that holds the mistake for the rest of the scene.

**Positional containment is not sufficient either.** Requiring the excerpt to
land in the final paragraph narrows the window without changing the property:
*"He thought again of the rations Bram had noted"* is a remote mention that sits
happily in a closing sentence. Proposing that as the fix repeats the exact error
it diagnoses — trading one incomplete check for another.

### 3.2 The fix: demote the verb list from decider to verifier

Today the grammar test is applied to the entire passage:

```1982:1982:everlore-server/worker/processors/generation.processor.ts
      if (hasSceneParticipationGrammar(phrase, rawNarrative)) return true
```

Scanning all the prose for a verb is **vocabulary deciding fiction** — banned by
§1. Apply the *same function* to only the judge's cited excerpt and it becomes a
check on the citation's shape — **verification**, which §1 permits:

```ts
// decider (banned): does this verb appear anywhere in the fiction?
hasSceneParticipationGrammar(name, rawNarrative)
// verifier (allowed): does the sentence the model chose to cite
// actually show this person participating?
hasSceneParticipationGrammar(name, citedExcerpt)
```

This resolves four problems at once:

- **It inherits the job Phase 1 would otherwise drop** — remote-mention is
  caught again, so the swap becomes safe.
- **It composes instead of competing.** The list stops arbitrating between two
  models and starts checking one model's work. That is the real defect the
  "one authority" principle was groping at.
- **The lists no longer have to die for Phase 1 to be safe.** They retire later,
  on their own schedule, at far lower risk.
- **Incompleteness becomes much cheaper.** A missing verb in a *decider* silently
  rules a present character absent. A missing verb in a *verifier* only rejects
  one candidate citation — and the model chooses which sentence to cite, so it
  can pick the most grammatically explicit one.

Stack positional containment on top: both checks then run over the citation
only, and no regex anywhere decides what happened in the fiction. Land this
**before** Phase 1 flips — on its own it strictly strengthens code already in
prod.

### 3.3 Unmeasured: the cutaway rate

The judge returns `present: []` whenever `player_viewpoint_at_end` is false.
Under Phase 1 that is a blanket admission denial on every cutaway ending —
fail-closed, consistent with principle #5, but the rate is unknown and this
product has cutaways. Not novel behaviour (`:1690` already empties the cast on
`!playerViewpointAtEnd` for breaks today), but it moves from the minority path
to the majority one.

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
reading events from Mongo.

**I called that "a task, not a phase." That was wrong** — not because the
plumbing is bigger than I said, but because there is nothing to run it over.
Measured against the live DB (`scripts/corpus-census.ts`, read-only):

```
events            233 total, 32 instances, avg 7.3 turns/world
                  span 2026-06-08 .. 2026-09-02
depth             1 world >50 turns | 2 worlds >20 | 7 worlds >=5 | 11 worlds =1
has ai_response   232  (99.6%)   ← prose exists
has player_input  177  (76.0%)
has scene_state     1  ( 0.4%)   ← first written 2026-09-02T10:27
```

`scene_state` exists on **one event, written today**, so it is newness rather
than a write bug — and it means a presence shadow-compare has *no history to
diff against at all*. The depth distribution is worse than the average: the
carry-forward bugs this work exists to kill only appear in long runs, and there
is exactly one world longer than fifty turns.

So the corpus has to be **generated** — `agent-chat` playthroughs, real tokens,
real hours. That is a phase, and everything downstream is blocked on it,
including the Phase 2 tier experiment. It is also the honest answer to "we are
not going live until we have stress-tested every single thing": today there is
nothing to stress-test against.

**The ledger itself is not empty, though.** A single sampled row read as
all-zero; aggregated, 92 of 197 rows (46.7%) carry a non-zero signal:

```
movement  detected 22 / committed  4     ← 18% commit rate, a real live gap
presence  detected 51 / committed 51     (all canon tier)
time      detected  1 / committed  1
party     detected  0 / committed  0     ← genuinely nothing
kinship   detected  0 / committed  0     ← genuinely nothing
miss_candidates 66 · player_corrected 7
```

Movement's 22-detected/4-committed split is exactly the disagreement signal I
claimed we would have to build shadow-compare to see. It is already there.

But the ledger still cannot measure Phase 1, for a sharper reason than
emptiness: **it is instrumented one layer away from the decision.**
`presenceTiers` is computed from `trackableMentions`
(`generation.processor.ts:3384-3389`), so presence `detected`/`committed` tracks
the *mention classifier*, not the corroboration gate Phase 1 replaces. Reaching
the arbitration outcome needs a new ledger field, not a new collection.

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
- **Scope the grammar verifier to the citation (§3.2).** Strictly strengthens
  code already in prod; deletes nothing; unblocks Phase 1.
- **Capture provider `usage` in `src/ai/client.ts`.** `generation_logs` records
  one `tokens_in`/`tokens_out` pair for the *narration* call and
  `metadata_model` as a bare string; there is no per-call accounting for any of
  the five post-stream calls, and `client.ts` never reads the provider's `usage`
  object. Until that exists Phase 2 can be measured for quality but **not for
  cost**, and no cost budget is enforceable.

### Phase 0.5 — generate the corpus  ← *the actual critical path*

Promoted out of Phase 1.5 after measurement (§4.3). 233 turns across 32 worlds,
one carrying `scene_state`, is not a corpus. Everything below is blocked on
this, including Phase 2. `agent-chat` playthroughs, deep runs over broad ones,
and persist raw witness/judge JSON via the existing `onRaw` hooks while
generating so the corpus is diffable rather than merely re-runnable.

### Phase 1 — consume the judge we already pay for  ← *start here*

Feed `adjudicateSceneEndpoint` into the cast on continuation turns, not only on
scene breaks. Its brief — *who is physically co-located with the player at the
final moment* — is already the correct question for a continuation.

- **Shadow first:** compute the endpoint cast on continuations, ship the current
  answer, log disagreement into `signal_ledger` (which already has the shape).
- **Then flip**, and `hasSceneParticipationGrammar` loses its job by
  construction: the corroboration gate becomes "the judge cited verified prose."
- Retires the `tierFor` ladder — `SPEECH_VERBS`, `ACTION_VERBS`,
  `PERSON_POSSESSIONS`, `POSSESSED_THING_ACTS_VERBS`, `TITLE_WORDS` — but only
  as *deciders*. Under §3.2 they survive as citation verifiers first and are
  narrowed or replaced later, separately.
- **Cost: zero marginal anything.** I originally hedged this with "possibly a
  modest `maxTokens` lift since continuations carry more names." That hedge was
  unnecessary and I was wrong to add it: the candidate list is assembled
  unconditionally at `:1251` from `priorPresent + choiceRoster +
  entityCandidates`, and `sceneBroke` does not exist until `:1611`, 360 lines
  later. **The judge's input is byte-identical on break and continuation turns
  today**, capped at 40 candidates in / 12 out on every turn. There is no larger
  candidate set to grow into.

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

### Phase 1.5 — finish the harness (small, once Phase 0.5 has produced data)

Extend `nsfw-extraction-probe.ts` from an inline fixture to a Mongo-backed batch
runner over stored `player_input` + `ai_response`; report through
`signal-ledger-report.ts`. Add a ledger field at the *arbitration* layer, not
the detector layer (§4.3). Re-extraction is non-deterministic, so this measures
distributions, not exact replay — fix a seed corpus and run n>1.

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

1. **One authority per question — or N readers with a declared, evidence-based
   tiebreak.** The defect was never two readers; it was that the *tiebreak* was
   a verb list. Amended after review: the original wording would have forced
   collapsing deliberate fault isolation (the witness was split on purpose) and
   defence in depth. Under the amended form both survive, and the regexes still
   die — because not one of them can be expressed as a declared evidence rule.
2. **Closed-world subcalls.** A judge selects from a supplied candidate set and
   cannot invent. Already true of both existing judges.
3. **Evidence-carrying and machine-verified — for a named property.** Every
   claim quotes the prose and the server verifies it. But state *which* property
   is verified: literal containment proves only non-fabrication (§3.1). Relevance
   needs its own check over the citation (§3.2). "Machine-verified" without
   naming the property is how the gap in `hasExactEvidence` went unnoticed.
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

## 9. Revisions after review

Recorded rather than quietly edited, because two of these change what to do
first and one of them was mine to catch.

| # | Claim as first written | Verdict |
|---|---|---|
| 1 | *"Computed, paid for, and thrown away"* | **Imprecise.** Only `present[]` is discarded; `playerViewpointAtEnd` + `sceneTransition` already feed `sceneBroke` at `:1618` every turn. Net-favours the plan — the judge is already trusted on continuations — but the framing was load-bearing and wrong. |
| 2 | *"Zero new calls plus a modest `maxTokens` lift"* | **Wrong in my favour.** Judge input is byte-identical on both paths (`:1251` vs `:1611`). Zero marginal anything. |
| 3 | *"Phase 1 is a superset swap"* | **Wrong, and the one I should have caught.** `hasExactEvidence` verifies non-fabrication, not endpoint relevance (§3.1). Fixed by scoping the grammar test to the citation (§3.2) rather than by positional containment, which narrows the window without changing the property. |
| 4 | *"Shadow-compare substrate exists — a task, not a phase"* | **Half wrong.** Plumbing does exist and the ledger is 46.7% populated, not empty. But 233 turns / 1 `scene_state` row means there is nothing to replay, so corpus generation is the critical path (§4.3). |
| 5 | *"One authority per question"* | **Too strong.** Amended to allow N readers with a declared evidence-based tiebreak, preserving deliberate fault isolation. |
| 6 | *"The arbitration framing is tidier than the git history"* | **Understated — it is retrofitted.** `git log -S`: `ACTION_VERBS`/`SPEECH_VERBS` landed `82b6d1c` **2026-06-19**; `adjudicateSceneEndpoint` landed `b666dd7` **2026-08-26**, ten weeks later. The lists were never arbitrating between judges — they were patching one unreliable witness. Which is exactly why Phase 1 deletes a check without inheriting its job (#3). The lone exception is `POSSESSED_THING_ACTS_VERBS` (`66ae987`, **2026-09-02**), added *after* the judge existed: a list genuinely chosen over an available judge. |
| 7 | Cost gates (*"≤ +15%"*) | **Unenforceable today.** No per-call token accounting for any post-stream call; `client.ts` never reads provider `usage`. Added to Phase 0. |

Diagnosis and phase *ordering* survived review. The one-line framing, the cost
arithmetic, and the superset claim did not.

---

## 8. Live state

Unchanged from DEVOCABULARY_PLAN §6: server `729ab07` is in prod, app 1.0.4
(vc7) is on the Play alpha (closed) track.

Nothing proposed here requires reverting either. Phase 0 is pure subtraction of
unreachable code; Phase 1 ships in shadow mode and changes no behaviour until
its disagreement log is understood. The widened lists in prod are better than
what preceded them and wrong by design — both remain true, and neither is
urgent enough to justify pulling a release.
