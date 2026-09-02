# Phase 0 + 0.5: the executable plan

**Status:** converged after two review rounds, Sept 2 2026. Ready to build.
**Supersedes** the *sequencing* in [DEVOCABULARY_PLAN.md](DEVOCABULARY_PLAN.md)
and [HARNESS_STUDY.md](HARNESS_STUDY.md). Their diagnoses stand; this is the
order and the acceptance criteria.

**Prod at time of writing:** server `58d6e72`, app 1.0.4 (vc7) on Play alpha
(closed). Nothing below requires reverting either.

---

## The rule

> A regex may **verify** or **normalize**. It may not **decide what happened in
> the fiction.**

One line, no carve-outs. Safety lists, closed enums, JSON repair, prose hygiene
and name normalization all pass it. `hasSceneParticipationGrammar` scanning a
whole passage for a verb does not.

## The shape of the work

Everything in Phase 0 **strengthens code already in production and deletes no
behaviour.** Nothing changes a narrative decision until Phase 1, which is gated
on measurement that does not exist yet.

Phase 0 is seven independent workstreams. Any can ship alone; none blocks
another. Phase 0.5 depends on 0.5 and 0.6 landing first.

---

## Phase 0.1 — Delete unreachable code

Pure subtraction, zero runtime behaviour change.

- Delete `worker/lib/scene-location-signal.ts` entirely. `establishesSceneLocation`
  is its only export and has exactly one reference: its own `export`.
- Delete from `worker/lib/movement-signal.ts`: `isGroundedPlayerDestination`,
  `refinePhysicalDestination`, `isExplicitPlayerLocationChange`, plus constants
  used only by them. **Keep `resolvePossessiveRoomName`** — live internally at
  `:540`.
- Drop the corresponding cases from `scripts/movement-signal-audit.ts`, which
  currently asserts against functions nothing calls.

**Accept when:** `bunx tsc --noEmit` clean, `audit:movement` green, no
remaining references.

## Phase 0.2 — Fix the false doc comment

`worker/lib/time-skip-signal.ts:10-11` claims *"a model-reported `time_elapsed`
always wins."* It never wins: `generation.processor.ts:1580` assigns
`narratedTimeLabel = playerTimeLabel` and the witness value only reaches the
signal ledger at `:3397`. The calendar is 100% regex.

Correct the comment to state what the code does. One-line change, high value —
the current text asserts a safety property that does not hold.

## Phase 0.3 — Audit runner + CI

There are **zero** test files and no test runner. `.github/workflows` contains
`deploy.yml` and nothing else. There are ~32 `bun` audit scripts, many with real
expected-vs-actual assertions, and nothing runs them together.

- Add `bun run audit:all` fanning out every pure-function audit (exclude the
  `LIVE=1` and DB-integration ones).
- Add a CI workflow running `tsc --noEmit` + `audit:all` on PR.
- **Seed it with the phantom-presence fixtures below** — the clearest
  expected-vs-actual pairs produced in this whole exchange, and nothing in the
  repo would currently catch a regression on them.

## Phase 0.4 — Capture provider token usage

`src/ai/client.ts` never reads the provider's `usage` object.
`generation_logs` has one `tokens_in`/`tokens_out` pair for the **narration**
call and `metadata_model` as a bare string — no per-call accounting for any of
the five post-stream calls.

Thread `usage` out of `callLLM` and record per-call. Until this exists, the
Phase 2 tier experiment can be measured for quality but **not for cost**, and
no cost budget (the "≤ +15%" gate in DEVOCABULARY_PLAN) is enforceable.

## Phase 0.5 — Persist raw extractor output  ← *blocks Phase 0.5 corpus*

`onRaw` is defined on all six extractors — endpoint judge at
`scene-endpoint-adjudicator.ts:51` — and subscribed by **nothing in
production**. The only subscriber is `scripts/nsfw-extraction-probe.ts`, which
does not even wire the endpoint judge's.

Subscribe the hooks and persist raw witness + judge JSON per turn
(fire-and-forget, off the critical path, same discipline as `signal_ledger`).

**This must land before corpus generation.** A corpus generated without citation
capture contains nothing to measure checks (b)/(c) against, and the only reason
corpus generation is a phase at all is that it burns real tokens and hours —
so getting this wrong means paying for it twice.

## Phase 0.6 — The citation verifier (advisory)

The substantive change. Today one check gates an endpoint presence claim
(`scene-endpoint-adjudicator.ts:110`):

```ts
if (!canonical || seen.has(key) || !hasExactEvidence(evidence, params.prose)) continue
```

`hasExactEvidence` runs `.includes()` over the whole passage, so it proves
**non-fabrication only** — the prompt demands proof of presence at the final
moment and the verifier accepts a quote from paragraph one.

### The stack

| | Check | Status | Property proved |
|---|---|---|---|
| a | excerpt appears verbatim in the prose | exists | not fabricated |
| b | excerpt contains a surface of the claimed name | **new** | the quote is about *this person* |
| c | excerpt shows **action** participation grammar for that name | **new** | acting, not referenced |

(b) is DEVOCABULARY_PLAN's, and it is a prerequisite for (c) — you cannot test
participation grammar on a citation that need not mention the person.

**(d) positional containment is dropped.** It narrows the window without
changing the property; a remote mention sits happily in a closing sentence.

### Why (c) is action-only

`tierFor` holds two kinds of pattern:

- **Action** — `SPEECH_VERBS`, `ACTION_VERBS`, `PERSON_POSSESSIONS`,
  `POSSESSED_THING_ACTS_VERBS`
- **Identity** — appositive, title-name, possessive kinship

All three identity patterns admit a dead or long-absent person to the scene cast
from a single passing mention. Verified against the live gate at `58d6e72`:

```
true   Rhea   The letter mentioned Captain Rhea, who had died at sea two winters before.
true   Mara   He still kept the locket. Mara, my sister, had been gone for years.
true   Mara   I never speak of my sister Mara any more, not since the burial.
```

That is the phantom-presence bug the gate exists to prevent, sitting inside the
gate. Identity evidence answers *"is this a person, and which one"* — which is
`adjudicateEntityCandidates`' job, a separate judge that legitimately reads the
whole passage.

### Narrow at the call site, not in the function

`hasSceneParticipationGrammar` has three production callers and the delegation
argument fails at one of them:

| Call site | Purpose | Mode |
|---|---|---|
| `generation.processor.ts:1983`, `:1990` | presence corroboration | **action-only** |
| `generation.processor.ts:2111` | `openingCast` seeding | **full (unchanged)** |

`:2111` reads `authored` — the seed event's prose, `model_used === 'seed'` —
while both judges run on `rawNarrative`. **No judge ever reads the authored
opening**, so there is nothing to delegate identity evidence to on that path.
Measured against real `world_templates`, action-only loses `Queen Isolde` from
*"Queen Isolde barely glances at you across the royal table"*, recreating
exactly the bug the `:2079` comment documents.

Implementation: add an options argument defaulting to today's behaviour, so
`:2111` is untouched by construction.

```ts
export function hasSceneParticipationGrammar(
  display: string,
  prose: string,
  opts?: { evidence?: 'all' | 'action' },  // default 'all'
): boolean
```

### Ship it advisory

(c) logs its verdict; **(a) enforces as today, (b) and (c) do not gate yet.**
Promote to enforcing only when captured citations show what it actually rejects.

This is not caution for its own sake — the adjacency measurement below says the
false-negative rate is materially higher than first estimated, and unknown.

### Per-check observability (required, not optional)

Log which of (a)/(b)/(c) rejected each citation, or the gate silently strangles
admission and it surfaces from a play-test weeks later. Precedent:
`presence.uncorroborated_held` (`generation.processor.ts:1941-1946`).

## Phase 0.7 — Fixtures for the phantom and adjacency cases

Add to a new `scripts/presence-evidence-audit.ts`, wired into `audit:all`:

**Phantom presence — assert action-only rejects, full admits** (documenting both
the fix and why `:2111` still needs `full`):

```
Rhea   "The letter mentioned Captain Rhea, who had died at sea two winters before."
Mara   "He still kept the locket. Mara, my sister, had been gone for years."
Mara   "I never speak of my sister Mara any more, not since the burial."
```

**Subject-verb adjacency — all verbs below are already in the lists:**

```
PASS  Isolde  "Queen Isolde barely glances at you across the royal table."  ← title-name rescue
FAIL  Bram    "Bram finally said nothing at all."
FAIL  Bram    "Bram, still furious, said nothing."
FAIL  Mara    "Mara almost nodded, then stopped."
```

**The finding these encode, which neither review round stated:** the pattern is
`\b${name}\s+(?:VERBS)\b` — strict adjacency, and one adverb defeats it. The
`Isolde` case passes the *full* gate only because `queen` triggers title-name.
So **the identity patterns have been masking the action patterns' adjacency
brittleness.** Removing identity does not merely narrow the gate, it exposes a
pre-existing weakness that was never visible. That is why (c) ships advisory,
and it makes objection 2's false-negative estimate a *span-length* red herring:
the loss is driven by pattern shape and applies at full-passage scope too.

---

## Phase 0.5 — Generate the corpus  ← *the critical path*

Strictly after 0.5 (raw capture) and 0.6 (advisory verifier), so every generated
turn carries citations and per-check verdicts.

Measured state of the DB today (`bun run report:corpus-census`, read-only):

```
events            233 total, 32 instances, avg 7.3 turns/world, span 2026-06-08..09-02
depth             1 world >50 turns | 2 worlds >20 | 7 worlds >=5 | 11 worlds =1
has ai_response   232  (99.6%)
has scene_state     1  ( 0.4%)   ← first written 2026-09-02T10:27
```

One `scene_state` row, written the same day, is not something to shadow-compare
against. And the depth distribution is worse than the average: carry-forward
bugs only appear in long runs, and there is exactly one world past fifty turns.

- Generate via `scripts/agent-chat.ts`. **Depth over breadth** — carry-forward is
  the failure mode.
- Target coverage: all three archetypes (GM / sentient / character), ≥3 genres,
  and enough long runs to see carry-forward — the ≥500-turn target in
  DEVOCABULARY_PLAN is a reasonable starting number, but *depth distribution* is
  the real acceptance criterion, not the total.
- Include the adversarial list from DEVOCABULARY_PLAN §4; every item is a real
  bug from this project's history.

**Accept when:** enough turns with captured citations to compute, per check,
what (b) and (c) would have rejected — including the cutaway rate
(`player_viewpoint_at_end === false` → `present: []`, currently unmeasured).

---

## What Phase 1 needs before it flips

Recorded here so the gate is not renegotiated later:

1. (b)/(c) rejection rates from real citations, classified new-correct /
   old-correct / both-wrong.
2. The adjacency false-negative rate at citation scope, measured not estimated.
3. The cutaway rate.
4. A ledger field at the **arbitration** layer. `signal_ledger`'s presence
   channel reads `detected 51 / committed 51` — a 100% commit rate, because
   `presenceTiers` derives from `trackableMentions`
   (`generation.processor.ts:3384-3389`). It measures the mention classifier, not
   the corroboration gate Phase 1 replaces, and cannot see Phase 1's subject.
5. TTFT unchanged (nothing here touches pre-stream), post-stream p95 not worse,
   and a cost delta that is now actually measurable via 0.4.

---

## Explicitly deferred

- **Movement / place lists.** `narratedArrival` requires a player-input regex and
  ignores `viewpoint_moved`; untangling that is its own phase.
- **Party.** Fix the privilege inversion first — it bypasses the corroboration
  gate via `travelKeys` (`scene-state.service.ts:244`, force-add `:269-275`,
  purge exemption `generation.processor.ts:1898`) — then add an evidence-carrying
  field. Ledger shows party `detected 0 / committed 0`: no data at all today.
- **Time.** A behaviour change, not a cleanup: the model has never had calendar
  authority. Last, own gate.
- **Model tier experiment.** Blocked on the corpus and on 0.4.

## Recorded constraints, not blockers

- **English-only.** The capitalisation concern was misattributed: `tierFor`
  matches `\b${name}\b` case-insensitively against a *supplied* name and never
  infers personhood from case. That dependence lives upstream in candidate
  discovery (`readsAsCommonNoun`, `appearsMidSentence`), which this plan does not
  touch. The real adjacent risk is that the **verbs** are English — but there is
  no language or locale field anywhere in `src/models` or `src/services` and no
  `l10n`/`i18n` directory, so the product is English-only today. Future
  constraint to record; does not gate Phase 0, and shipping (c) advisory removes
  the fail-closed exposure entirely.
- **Aspect is the ceiling on this family.** None of the seven patterns encode
  tense, so *"Bram said it would rain, three days ago in the valley"* confirms
  Bram present today. The canonical *"the rations Bram had noted"* case is caught
  only because `noted` is absent from the list and `had` sits between name and
  verb — luck, not design. Aspect and argument structure are *structural*
  properties, testable without deciding which verbs count, which is where the
  lists eventually retire to. Nobody has verified that is tractable here.
