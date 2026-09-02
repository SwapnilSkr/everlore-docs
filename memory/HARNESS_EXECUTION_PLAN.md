# Phase 0 + 0.5: the executable plan

**Status:** Phase 0 + 0.8 + **Phase 1** + **Phases 3/4/5 landed locally** (not
prod). Continuations admit newcomers from endpoint `present[]` (`(a)∧(b)∧(c)`);
prior cast still carries. The cursor now has a second, independent namer and the
same citation stack; travel privilege is scoped to the scene break; the calendar
accepts a cited narrator skip. Model-tier remains deferred. See
[§ Phases 3–5](#phases-35--place-party-time-landed-locally) at the end.
**Supersedes** the *sequencing* in [DEVOCABULARY_PLAN.md](DEVOCABULARY_PLAN.md)
and [HARNESS_STUDY.md](HARNESS_STUDY.md). Their diagnoses stand; this is the
order and the acceptance criteria.

**Prod at time of writing:** server `58d6e72`, app 1.0.4 (vc7) on Play alpha
(closed). This work is **not** going to prod until the gates below hold.
Local verification used the emulator (Aurelius Valemont, 3 turns) plus
`agent-chat` on three new worlds.

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

**Do not add a sixth post-stream call.** The study's point still holds: we
already buy `adjudicateSceneEndpoint` on every turn and discard `present[]` on
continuations. The waste measured on Sept 2 was not a missing model — it was
paying mini for a citation and then dropping it because the *verifier* could
not read wrapping quotes, or because `ACTION_VERBS` demanded strict adjacency.
Harmony means the judge's prompt and the verifier prove the same property, so
a well-formed claim survives. It does not mean a new LLM, a new schema, or
replacing the five-call tail.

**Do not break:** openingCast `:2111` (full grammar on authored seed), input
guard / NSFW lists, JSON repair, memory supersession when the player corrects
canon, party/travel privilege (deferred), calendar authority (deferred),
pre-stream / TTFT path.

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

### Measured — Sept 2 2026 (local Phase 0 stack, not prod)

Capture works. Every generated turn wrote `extractor_raw` (including
`scene_endpoint`) and `generation_logs.llm_calls`.

| World | Turns | Citations | Notes |
|---|---|---|---|
| Aurelius Valemont (emulator, existing save) | 3 | 4 | Connected to local `:8081` |
| Vesperkeep Hall (GM, grimdark) | 14 | 16 | Adversarial list from §4 |
| The City That Watches Back (sentient, noir) | 12 | 13 | Modern / Gregorian |
| Reese After Soundcheck (character) | 12 | 11 | First-person companion |

**44 citations across 41 turns** (4 turns cited nobody).

| Check | Pass | Fail | Live meaning |
|---|---|---|---|
| (a) excerpt in prose | 75.0% | 25.0% | Mix of real non-quotes **and wrapping-quote / punctuation false fails**. This already gates. |
| (b) excerpt names this person | 56.8% | 43.2% | Pronoun citations (`he says`, `I watch you`). Character-lane `I` never contains `Reese`. |
| (c) action grammar on excerpt | 27.3% | 72.7% | Cannot enforce. See combos. |
| Cutaway (`viewpoint_at_end=false`) | — | **14.6%** (6/41) | Now measured. |

Combos:

- 11 all-pass (clean: `Aldric's jaw tightened`, `Tomas watched the exchange`)
- 12 (a) ok, (b)+(c) fail — quote real, not about that name (`Cedric` ← `the prince stood straighter`)
- 10 (a)+(b) ok, (c) fail — **the Phase 1 question**
- 11 (a) fail — already dropped today
- 0 (b) fail while (c) passes

Those 10 name-valid (c) fails are two classes, and that is why (c) does not
flip:

- **Correct rejects:** `You'll want to see Bram's numbers next.` / `the things Tomas didn't say outright.`
- **False rejects (adjacency / auxiliary / participle / relative):** `Tomas hasn't moved from the center of the room.` / `Tomas standing by the cold hearth.` / `Cedric, who stood silent and pale by the window.` / `Soren's inside. He's pouring…`

Among citations that already proved the quote is about that person, (c) fails
about half the time, and a large slice of those people are in the room.
Enforcing it would strip Tomas/Cedric/Soren from scenes they occupy. That is
the opposite of the phantom-presence fix, and it is the adjacency hole the
fixtures in 0.7 encoded, now seen live.

**Character / sentient first-person** is a separate failure class: the
endpoint judge cites `I watch you` for Reese. (b) is right to reject that
span; the model quoted the wrong evidence. Fix the prompt (name-first main
clause), do not teach the regex that `I` means Reese.

Instance ids (keepers, do not delete): emu `6a97d9c046932234f6c6ed48`;
GM `6a9838c242bd67e3f97250da`; sentient `6a9838c442bd67e3f97250e1`;
character `6a9838c742bd67e3f97250e8`.

---

## Phase 0.8 — Make the verifier match the rule

The rule is unchanged: a regex may **verify** or **normalize**; it may not
decide what happened in the fiction. (c) today still decides fiction by
asking whether a verb from `ACTION_VERBS` sits next to the name. That is the
list we are retiring. The live FNs are almost all *clause shape* (aux,
adverb, participle), which is verifiable without naming actions.

1. **Normalize (a).** Strip wrapping quotes, unify curly/straight apostrophes
   and quotes, keep `.includes()` as a fabrication check. Regex as normalize.
   This is live-gating today; a 25% fail rate that includes punctuation is
   dropping real people.
2. **Citation-scoped (c) becomes subject-predicate shape**, not a verb list.
   The excerpt must start with the claimed name (optional one-token title) and
   then a predicate head, skipping only a closed class of English function
   words (aux, negation, common adverbs). Reject `, who` relatives and
   `, my/the/a` identity appositives (those are the phantom-presence paths).
   Reject `'s` unless it is a locative copula (`Soren's inside`). Keep the
   existing body-part possessive path as a closed enum, which already
   verifies `Aldric's jaw tightened` without scanning for `nodded`.
   `hasSceneParticipationGrammar` at `:2111` (openingCast) stays `'all'`.
   Corroboration at `:1983`/`:1990` is unchanged in this phase — still
   advisory-adjacent, still not the endpoint gate.
3. **Prompt the endpoint judge** to quote a name-first main clause at the
   final moment. Pronoun spans and relative-only spans become the model's
   miss, not the verifier's.
4. **Replay the 44 captured citations offline** through the new verifier
   (free; the prose is in `extractor_raw` / events). Gold: (c) must pass the
   aux/adverb/participle presents and still reject Bram's numbers / dead
   Rhea / sister-gone. Do not promote (c) to enforcing until that replay
   plus a new live sample show **<10% false rejects on name-valid present
   people** and **no new phantom admits**.
5. **One deep run (≥50 turns, one world)** before any Phase 1 flip.
   Carry-forward was the original acceptance criterion; 12-turn worlds do
   not see it.

**Accept when:** `audit:presence-evidence` encodes the live FNs as now-passing
structural cases and the phantoms as still-failing; offline replay of the
Sept 2 corpus is classified; a new live sample (emulator + `agent-chat`)
does not reintroduce phantom presence.

### Offline replay — same 44 citations, new verifier (free)

No new LLM spend. Prose from the captured events; `evaluatePresenceCitation`
re-run locally.

| | Live Sept 2 (old verifier) | Replay (0.8) |
|---|---|---|
| (a) pass | 75.0% (33/44) | **95.5% (42/44)** — 9 wrapping-quote / curly-apostrophe rescues, **0 regressions** |
| (b) pass | 56.8% (25/44) | **52.3% (23/44)** — two "The City" rows lost the article-token false match (`The neon` / `the glass`). Correct. |
| (c) pass | 27.3% (12/44) | **43.2% (19/44)** — 7 structural rescues, **0 regressions** |
| name-valid (a+b) but (c) fail | 10 | **4**, all correct rejects |

**(c) rescues (were live false rejects):** `Tomas hasn't moved` · `Tomas standing by` · `the steward agreed to walk` · `Soren's inside` · `Soren's behind the bar` · wrapping-quoted `Soren freezes` · `the city feels it`.

**Remaining 4 name-valid (c) fails — keep failing:**

| Excerpt | Class |
|---|---|
| `Cedric, who stood silent and pale by the window` | relative — prompt must quote a main clause |
| `the things Tomas didn't say outright` | name is not the clause subject |
| `You'll want to see Bram's numbers next.` | possessed ledger, not the person in the room |
| `Bram's down there now, with his ledgers.` | distal locative — named, not at the player's endpoint |

Among people the quote already named, structural (c) no longer false-rejects
the aux/adverb/participle/locative presents. The leftover fails are the
phantom-presence paths. **Still do not enforce (c)** until a new live sample
confirms the prompt actually emits name-first main clauses (so relatives and
pronouns become the model's miss, not a silent strip).

Character-lane `I watch you` for Reese is unchanged: (b) correctly fails;
do not teach the regex that `I` means the companion.

### Live sample after worker restart (still not prod)

Four new turns on the keeper worlds, worker process picked up 0.8.
Emulator Aurelius was not re-driven (different Google account than the OTP
harness). These are the agent-chat turns.

| World | Seq | Citation | (a)(b)(c) | Reading |
|---|---|---|---|---|
| Vesperkeep | 16 | `Tomas hasn't moved.` | all pass | The Sept 2 false reject, live, paid and kept. |
| Vesperkeep | 17 | `Tomas's eyes don't leave the cold hearth.` | all pass | Curly apostrophe + body-part possessive. |
| Vesperkeep | 17 | steward ← `He stays perfectly still…` | a pass, b+c fail | Prompt still emitted a pronoun for the wrong candidate. (a) would *admit* this on a scene break. This is why `present[]` stays continuation-discarded and why **(b) is the next candidate to enforce, not (c)**. |
| Split Lamp | 14 | `Soren just needs something to do with his hands.` | a fail, b+c pass | Paraphrase. Prose said "a man just needs…". (a) correctly dropped a fabricated span. Do not weaken (a). |
| Reese | 14 | `present_at_end: []` | — | First-person companion, nobody else in the room. Did **not** cite `I` as Reese. |

No new phantom admits. The one paid-then-dropped row is a real fabrication.
The one remaining harmony miss is a pronoun attributed to the steward; the
verifier already flags it; consuming that `present[]` on a continuation
would still be wrong.

The prompt was tightened after seq 17: a pronoun sentence must not be
attached to a different candidate; omit rather than guess. (b) is still
advisory. Scene-break admits that would be `(a) && !(b)` now log
`presence.citation_name_mismatch` (same shape as `presence.uncorroborated_held`)
so a deeper sample can say whether (b) is safe to enforce without reading
Mongo by hand.

Deep run in progress on Vesperkeep (keeper `6a9838c242bd67e3f97250da`),
adversarial list from DEVOCABULARY_PLAN §4, toward ≥50 turns on one world.
Phase 1 still does not flip.

### Deep run — Vesperkeep seq 18–43 (same keeper)

24 new turns on one world. Hall carry-forward (18–32) held: location stayed
`the hall`, cast stayed `Tomas`, Bram/Mara/Sister never entered the scene
state. Grab-sleeve did not enrol a companion. Asking about cellars without
walking did not move the cursor.

**Paid-then-dropped classes this run actually hit:**

| Seq | What happened | Class |
|---|---|---|
| 19 | `Tomas's weary gaze` — (c) failed on one adjective between `'s` and `gaze` | Fixed: citation (c) allows one modifier before a `PERSON_POSSESSION_TOKENS` word. Detector / corroboration lists unchanged. |
| 20, 27 | Verbatim span ended `.` while prose continued `,` — (a) dropped Tomas | Fixed: `hasExactEvidence` strips trailing sentence punct. Normalization, not a fiction-decider. |
| 20, 29 | `Bram's down there` / `Bram's numbers` — (a)(b) pass, (c) fail | Correct. Distal locative / possessed ledger. Do not add `down` to locative copula. |
| 23 | candidate `Sister` ← `your sister isn't coming back` — (c) fail | Correct. |
| 31 | steward ← pronoun span; logged `presence.citation_name_mismatch` | Prompt still missed once. (b) still advisory. |
| 33 | Walk to cellars alone. `broke=true`, empty cellar, Bram not invented | Correct. Endpoint `playerViewpointAtEnd=false`. |
| 38 | Return to hall. Judge cited Tomas `(a)(b)(c)` all pass. Cast still `[]` with `uncorroborated_arrival` | The verb-list corroboration undid a paid scene-break admit (`Tomas hasn't moved` fails `ACTION_VERBS` adjacency). **Fix:** on scene break, names whose endpoint verdict is `(a)∧(b)∧(c)` seed `proseCorroborated`. Continuations still discard `present[]`. Opening-cast `:2135` untouched. |
| 43 | Same return, after the seed. Raw JSON cited `Tomas hasn't moved.` `(a)(b)(c)` would pass offline, but `endpointPresent=[]`, `citation_verdicts` empty, cursor stuck in `root cellars` (`narratedMove=true`, `validDest=false`) | Separate from 38. Witness had Tomas (`primaryOnly`) and was discarded because scene-break trusts empty endpoint `present[]`. Location follow is the deferred movement/`narratedArrival` work — do not grow those lists here. |

Hall quiet-listen turns did not flicker Tomas out. That is the carry-forward
the 50-turn gate was for, and it held for 15 continuous-scene turns before
the first walk.

Do not consume `present[]` on continuations. Do not enforce (c) as a live
gate. The scene-break seed is the minimum so a citation we already paid for
and already verified cannot be stripped by the list it is supposed to retire.

---

## Harmony — what to improve vs what not to break

The study's inventory still holds: **five** post-stream `gpt-4o-mini` calls
in one `Promise.all` (`generation.processor.ts:1278`) — witness, choice-meta,
entity judge, deaths, endpoint. Two of the endpoint's four fields already
feed `sceneBroke` every turn. `present[]` is consumed only when the scene
breaks (`:1904`). That topology is load-bearing. Replacing it, or adding a
sixth call so the verifier "agrees", is the tomfoolery the study warned
against.

**Harmony means the judge and the verifier prove the same property**, so a
well-formed paid claim survives. It does not mean a new model, a new schema,
or growing `ACTION_VERBS`.

### Improve (0.8 / this work)

| Surface | Change | Why this is not a fiction-decider |
|---|---|---|
| `comparable()` / (a) | Strip wrapping quotes, unify curly quotes | Normalization. (a) already gates; punctuation was dropping real people. |
| Citation-scoped (c) | Subject-predicate *shape* + locative `'s` copula; keep body-part list as the existing closed enum | Verifies the excerpt's clause shape. Does not consult `ACTION_VERBS`. |
| Endpoint prompt | Name-first main clause; never cite `I`/`he`/`she` as a name | Same property the verifier checks, so the model is asked for what we can keep. |
| Fixtures + replay | Encode live FNs and phantoms | Measurement, not a runtime gate. |

Post-stream correction of a later turn is still the product: if turn N states
something and turn N+1 corrects it, memory supersession and scene-state
carry-forward already handle that. Do not add an LLM "agreement" pass on top.

### Do not break (leave these call sites and behaviours alone)

| Site | Why |
|---|---|
| Opening-cast seeder `generation.processor.ts:2135` — `hasSceneParticipationGrammar(surface, authored)` with default `'all'` | Authored seed has no judge. Identity patterns must still admit "Your sister Neva leans against the hearth". Changing this re-opens the sister-written-out-of-the-hall bug. |
| Corroboration `generation.processor.ts:2007` / `:2014` | Full grammar still backs openingCast and the outage path. New admits on the happy path are endpoint `(a)∧(b)∧(c)`, not this list. |
| Quiet-cast carry-forward | Continuations still keep `priorPresent`. Do not replace the whole room with endpoint-only names. |
| `playerViewpointAtEnd` + `sceneTransition` on every turn (`:1642`) | Already trusted. Do not gate these on (b)/(c). |
| Identity half of `hasSceneParticipationGrammar` | Stays in the function. Citation (c) does not call it with `'all'`. Do not delete the patterns; openingCast needs them. |
| `ACTION_VERBS` / `PERSON_POSSESSIONS` in `presence-gap-detector.ts` | Still the corroboration / openingCast / listed-action backstop. Do not grow them to paper over adjacency. Do not delete them in 0.8. |
| Input guard / NSFW lists, JSON repair, prose hygiene | Safety and parse, not fiction. |
| Memory supersession when the player corrects canon | Post-stream correction path. Untouched. |
| Party / travel privilege (`travelKeys`, force-add) | Deferred; privilege inversion is a separate bug. |
| Calendar authority | Player-input regex is authoritative; witness `time_elapsed` is ledger-only. |
| Pre-stream / TTFT | Nothing here may wait on post-stream. |
| A sixth post-stream call | The waste was verifier≠prompt, not a missing completion. |

### Paid-then-dropped after 0.8 — remaining classes, still not a new LLM

1. **Continuation `present[]`.** Still computed and discarded. That is Phase 1,
   after the prompt actually emits name-first clauses on a live sample.
2. **Pronoun / first-person evidence.** (b) is right. The model quoted the
   wrong sentence. Prompt, not regex-as-Reese.
3. **Relative-only evidence** (`Cedric, who stood`). Verifier correctly
   rejects; the prompt forbids it. If live turns still emit these, that is a
   prompt miss, not a reason to admit `, who`.
4. **Witness vs endpoint on continuations.** They are allowed to disagree
   until Phase 1. Do not reconcile them with another model.

---

## What Phase 1 needs before it flips

Recorded here so the gate is not renegotiated later. **First measurement
(Sept 2, n=41 turns / 44 citations) is in. It is enough to refuse the (c)
flip; it is not enough to ship (b) as a gate.**

1. (b)/(c) rejection rates from real citations, classified new-correct /
   old-correct / both-wrong. **Replay:** name-valid (c) fails are 4/44 and
   all correct rejects (relative / remote / possessed-NP). (b) fails remain
   mostly pronoun / first-person spans, plus two sentient-name article
   matches that 0.8 now correctly refuses. **Still do not enforce (c).**
2. The adjacency false-negative rate at citation scope, measured not estimated.
   **Measured:** aux (`hasn't moved`), adverb (`finally said` in fixtures),
   participle (`standing by`), relative (`who stood`). Live, not hypothetical.
3. The cutaway rate. **14.6%** (6/41).
4. A ledger field at the **arbitration** layer. **Landed.** Presence
   `detected` is endpoint citations; `committed` is how many of those names
   are in the final scene cast. Cutover on new rows only.
5. TTFT unchanged (nothing here touches pre-stream), post-stream p95 not worse,
   and a cost delta that is now actually measurable via 0.4. **Plumbing is
   live** (`llm_calls` on all 41 turns); no A/B yet.

**Phase 1, landed locally (not prod):**

Continuation candidates are `priorPresent + endpoint present[] + party`, not
`prior + witness`. Quiet people still carry. New admits need `(a)∧(b)∧(c)` —
verbatim, names this person, subject-predicate / body-part — which is the
property the judge already quotes. Witness `present_characters` is the outage
fallback only. Opening-cast full grammar is unchanged.

`signal_ledger.presence` now records endpoint citations detected vs names from
those citations that landed in the scene cast. `by_tier` is the citation stack
(canon = a∧b∧c, hint = verbatim but not name-acting, hidden = (a) fail).

Live check (Split Lamp seq 15–17, after the worker picked up Phase 1):

| Seq | Player | Cast | Reading |
|---|---|---|---|
| 15 | Don't name anyone | Soren | Endpoint cited nobody (`He watches`). Carry-forward kept Soren. |
| 16 | Someone who is not in this room | Soren | Homework girl did not enter the cast. |
| 17 | Look at Soren | Soren | Citation failed (a) (paraphrase). Carry-forward kept him anyway. |

Phases 3–5 (movement, party, time) and the model-tier experiment are still
deferred: they are other pipelines, not this presence flip.

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


---

## Phases 3–5 — place, party, time (landed locally)

Same rule, same stack, three more pipelines. No sixth post-stream call: the
location arbitration is a new **field** on the endpoint judge we already buy.

### Phase 3 — the location cursor

The cursor had exactly one check, `hasGroundedWitnessLocationEvidence`, which is
(a) alone. (a) is a fabrication check: it accepted *"the low road to Marrow
Ford"* as proof of arriving at Marrow Ford and *"Bram's down there in the
cellars"* as proof the player was in them. Its only corroborator was
`detectNarratedMovement` — a locomotion-verb list scanned over the player's text.

**1. `worker/lib/location-citation.ts` — the (a)(b)(c) stack for a place.**
(b) the excerpt names this place, every distinctive word of it. (c) the excerpt
situates the *viewpoint* at it, in one of three shapes: a locative PP with no
competing subject, a clause the viewpoint owns, or the place as clause subject.
Tested per clause, so a subordinate clause about somebody else cannot poison the
main clause. The only lists are **closed grammatical classes** — locative
prepositions (`to`/`for`/`toward`/`from` deliberately excluded, they are goal
markers), determiners, viewpoint and third-person pronouns — plus the caller's
own cast names.

**2. The corroborator is tiered, not swapped.** A claim that passes the whole
stack stands alone: a merely-mentioned place cannot produce an excerpt that is
verbatim, names it, and situates the viewpoint at it. A claim with only (a)
still needs a second signal, and that signal is `judgeTransition ||
detectNarratedMovement`.

> The judge alone was tried first and **regressed a live turn**. Measured over
> the corpus: endpoint `scene_transition` is true on **5/81** turns while the
> witness reports `viewpoint_moved` on **14/81**. It under-reports real moves
> ~3×. The verb list stays as the other half of the disjunction — with strictly
> weaker authority, since it can now only ADD a move, never veto one.

**3. The endpoint judge gained `location_at_end` — a second, independent
namer.** This was the substantive discovery. The witness is instructed to hold
the prior location unless the viewpoint moved, and it obeys past the point of
truth: on a live run it reported the bar the player had left while the prose had
them standing on a canal bridge, for four consecutive turns. **A cursor cannot
follow a place nobody names.** Presence solved this with a second namer years of
bugs ago; location never had one.

**4. `passageSituatesViewpoint` — verify the claim against its own source.**
The judge picks the wrong sentence often: it named "the dock" correctly and
cited *"I lower myself to sit a few feet from you"*, while the same passage
contained *"I stay leaning against the brick wall beside the open dock door"*.
Searching the source for the property is verification, not invention — the model
chose the place, and a mentioned place cannot produce a qualifying sentence. It
subsumes (a).

**5. `PLACE_STEMS` is retired for names already on the map, kept for new ones.**

> **Placehood is not structural, and this cost a live turn.** Furniture takes
> exactly the grammar a room does — *"I sit on the old bench"* and *"I wait in
> the hall"* are the same shape. With the vocabulary bypassed on any proven
> citation, the cursor moved `the hall → the old bench` on a perfectly valid
> (a)(b)(c) citation. A name the map already knows is now admitted on its
> citation alone; a label new to the world still clears the vocabulary. The
> judge's prompt also names furniture explicitly as *not* a place, and a refused
> label logs `location.judged_name_not_a_place`.

**Do not tell the judge the known places.** Tried, reverted, measured: supplying
them (with the current cursor at the head) made the judge anchor on the stale
name and it started returning "the green room" for a scene on a loading dock.
Both models anchor when given a prior. The judge is told to read this passage
only and is not told where the scene started.

### Phase 4 — party privilege inversion

Two changes, both in the same direction: party membership was the one path into
a scene that answered to nothing the narration said.

- **`scene-state.service.ts` — privilege is scoped to the break.** `travelKeys`
  bypassed the corroboration gate and force-added on *every* turn, so a
  companion detected once rode along for the rest of the run, immune to the
  prose, until an explicit parting phrase fired. The privilege exists for one
  moment — a scene break wipes the prior cast, so a companion just ridden in
  with has nothing to carry forward. On a continuation the prior cast carries
  anyway, so the bypass bought nothing and only suppressed the evidence.
- **A free-form join must be corroborated.** A phrase match on the player's text
  used to be enough to enrol somebody for the whole run. It now also needs the
  endpoint judge to cite them or the entity judge to show them acting. The
  structured travel control is exempt — that is the player operating the
  product. Refusals log `party.uncorroborated_join_refused`.

**Live:** `"Soren, come with me — I want to walk the canal"`. Soren never left
his bar (`"He's still behind the bar"` two turns later). The join was refused.
Under the old code he would have been a travelling companion for the rest of the
run.

### Phase 5 — the calendar

`time-skip-signal.ts` documented the witness label as telemetry, and it was:
`narratedTimeLabel = playerTimeLabel`, a regex over the player's text, 100% of
the calendar's authority. A skip only the narrator wrote left the date untouched.

`time_evidence` is now a required witness field whenever `time_elapsed` is set,
verified by `worker/lib/time-citation.ts`: (a) verbatim in the narration,
(b) the excerpt carries the span the label claims, unit-matched — *"three days"*
may not be cited by *"three hours"*. There is no (c): whether time passed in the
fiction is precisely the model's judgement; what a citation can verify is that it
did not invent the sentence or inflate the span. The player's label still wins.
The stale doc comment is corrected.

### Corroboration is structural now, not a verb list

`showsParticipationInPassage` — the citation-scoped (c) test applied per
sentence — is the first path in `proseCorroborated`, and the **identity half of
`hasSceneParticipationGrammar` is no longer consulted there** (`evidence:
'action'`). Whole-passage identity matching is how *"Mara, my sister, had been
gone for years"* corroborated Mara's arrival. `openingCast` keeps the full
grammar: the authored seed has no judge to delegate identity to.

### Measurement

- `signal_ledger.movement.detected` now counts every arbiter's claim, not just
  the travel control and the witness booleans. It was recording `detected 0 /
  committed 1` for cursor moves admitted by the citation stack.
- New logs: `location.transition_disagreement`, `location.judged_name_not_a_place`,
  `party.uncorroborated_join_refused`, `time.witness_citation`, and the
  `location.decision` row now carries both citation verdicts and the judged name.

### Audits

`audit:location-evidence` (21 cases) and `audit:time-evidence` (11) are new;
`audit:presence-evidence` gained 12 structural-corroboration cases;
`audit:scene-state` gained 4 party-privilege cases. `bun run audit:all` is
**25 audits, all green**, `tsc --noEmit` clean.

### Live verification (local `:8081`, not prod)

~30 turns across three keeper worlds (grimdark GM, noir sentient, first-person
character). Highlights:

| What | Result |
|---|---|
| Cursor stuck in `root cellars` since seq 43 | Moved on the citation stack alone — `narratedMove=false`, `judgeTransition=false`, witness `viewpoint_moved=false`. Every legacy signal said no. |
| Bar frozen for 4 turns while the scene was on a canal bridge | Judge named `the canal`, cited it, cursor followed |
| `"the dim light of the loading dock"` | Judge named `the alley`, own citation failed (c), passage verification carried it |
| Absent brother discussed at length | Never entered the cast |
| NPC silhouette `"against the deeper black of the gatehouse road"` | (c) refused — the phantom-travel class, caught |
| `"the cellar"` while the cursor read `root cellars` | (c) refused, no duplicate node |
| `"The old bench groaning under him."` | (c) **passed** — this is the furniture bug above; now gated on placehood |
| 6 consecutive stay-put turns naming other places | Cursor held |

### Honest residuals

1. **`scene_transition` is 5/81.** The judge under-reports transitions badly.
   The verb list cannot retire until that is fixed or the location field proves
   sufficient on its own.
2. **Placehood is still a vocabulary** for labels new to the world. There is no
   structural test that separates a room from a bench, and the bench turn proves
   it is load-bearing. This is the one remaining list that decides fiction, it is
   scoped as narrowly as it can be, and it needs a different idea, not a wider
   list.
3. **Party corroboration asks "is this person here", not "did they come with
   you".** Strictly stronger than a phrase match, but not the real question. The
   real answer arrives on the next turn: a companion is whoever is present after
   the player moves.
4. **`memory.service.ts` replay** duplicates the old location logic and does not
   run the citation stack. Edit-replay can still move a cursor on (a) alone.
5. **`trackableFamilyLabels`** (36 role words in `generation.processor.ts`)
   decides whether a lowercase label is a trackable person, and has a documented
   contract with the Dart client and an audit. Untouched, still a violation.
6. **Calendar coverage is thin.** The witness sets `time_elapsed` on 2/81 turns.
   Correct for continuous scenes, but nobody has checked whether long runs
   actually advance dates.


---

## The replay corpus — the instrument, not another fix

Every judgement call in this work has been argued and then, when a real sample
was finally replayed, sometimes falsified. That happened three times in one
afternoon. The corpus exists so that stops being expensive.

```
bun run corpus:freeze     # read-only Mongo → corpus/turns.json (committed)
bun run corpus:replay     # FREE. current verifiers over what prod already returned
bun run corpus:tier       # COSTS MONEY. re-runs the extractors at N model tiers
```

`corpus/turns.json` is **311 turns across 24 worlds**, two of them past 60 turns,
with the prose, the player's text, and the exact context the extractors were
handed — prior location, prior cast, prior physical facts, known places, roster,
protagonist. 110 carry the extractions production actually returned.

It is committed. Freezing it is the point: a corpus that has to be regenerated
before every comparison is not a baseline.

### What `corpus:replay` says today (free, 311 turns)

| Presence citations | |
|---|---|
| (a) verbatim | 96.4% (106/110) |
| (b) names the person | 77.3% |
| (c) acting | 68.2% |
| **survives (a∧b∧c)** | **64.5%** |
| cutaway rate | 16.4% |
| `scene_transition` | 10.0% |

Presence is in good shape — that is up from **(a) 75% / (c) 27.3%** in the Sept 2
sample, which is the 0.8 normalization and structural (c) working.

| Location claims | |
|---|---|
| (a) verbatim | 91.8% |
| (b) names the place | 19.1% |
| (c) situates the viewpoint | 15.5% |
| passage supports the claim | 23.6% |
| held the prior place | 91/110 |
| …and the passage does not support it | 74/91 |
| …**and the passage puts them in another KNOWN place** | **0/91** |

That last row is the honest correction to my own headline. "Unsupported" is not
"wrong": a quiet continuation names no place at all, and the cursor is right to
sit still. On places the world already knows, the witness's anchoring is **never
demonstrably wrong on this corpus.**

The anchoring failure that was observed live (a bar held for four turns while
the prose was on a canal bridge) involved a place that was **new**, which this
test cannot see by construction — it has no way to enumerate candidate places
without a place vocabulary. That is precisely what the tier experiment
adjudicates head-to-head, because two tiers naming different places generate the
candidates for free.

### On adjudicating without ground truth

`corpus:tier` scores what can be scored automatically — schema validity, whether
the passage situates the viewpoint at the claim, citation survival, tokens,
latency — and prints every location disagreement with two independent flags
(`passage`, `onmap`) for a human to read.

It deliberately does **not** declare a winner from passage support alone. The
very first disagreement it produced was `"the royal council chamber"` (gpt-4o-mini)
against `"at the table"` (gpt-4o and gemini) — and passage support prefers *the
table*, because furniture takes a room's grammar. That is the same finding as
the `the hall → the old bench` cursor move, arriving from a completely different
direction on the same afternoon. **Placehood is the load-bearing unsolved
problem in this system**, and any automatic metric that ignores it will rank
models backwards.


---

## Ground truth, and the three defects the instrument found in itself

`corpus:gold` labels each sampled turn with a strong model reading the full
passage plus the pipeline's own context, and quoting the sentence it decided on.
`corpus:accuracy` scores production against it. Both free to re-run.

**Three measurement defects surfaced in one afternoon, each of which would have
sent months of work in the wrong direction. None was findable from a playtest.**

1. **Gold refused to carry forward.** It was told not to inherit the prior place,
   so a correct carry ("still in the Split Lamp") scored as an *invented* place.
2. **The sampler was round-robin over 24 worlds**, most of them 1-8 turns long.
   Median sampled sequence: **4**. The corpus contains 73- and 65-turn runs and
   was scoring almost none of them.
3. **`context.priorLocation` came from `data.scene_state`, first written
   2026-09-02.** On historical turns it was null, so the entire corpus was a
   COLD-START benchmark: every extractor asked to establish a location from
   nothing, on every turn. `location_anchor` has been written all along and
   covers 252/346 turns. Fixed: prior location known on **237/311**.

Median sequence went 4 → 13 (p75 26, max 54); prior location 6/50 → 31/50.

### Location + cast accuracy, corrected corpus, 50 turns

| | gpt-4o-mini (production) | gpt-4o |
|---|---|---|
| RIGHT — named the place | 32 (64%) | 32 (64%) |
| **HARMFUL — bad write** | **17 (34%)** | **9 (18%)** |
|   named the wrong place | 5 | 4 |
|   invented one | 12 | 5 |
|   …furniture | 2 | 2 |
| LOST — no write, cursor holds | 0 | 1 (2%) |
| SAFE — nothing to say | 1 | 8 (16%) |
| CAST exact match | 37 (74%) | 35 (70%) |
|   precision (phantoms) | 83.3% | 92.5% |
|   recall (dropped) | 86.2% | 63.8% |

**The tier story is not what the cold-start run said.** Given real prior context
the two models name the place correctly *equally often* (64%). mini is even
slightly better on exact cast match (74% vs 70%). The entire difference is what
they do when unsure: **mini converts uncertainty into a harmful write, gpt-4o
converts it into an abstention** — 34% vs 18% harmful, and 1 vs 8 safe nulls.
On cast, mini trades precision for recall (83/86) and gpt-4o the reverse
(93/64). Under carry-forward a dropped name recovers next turn and a phantom
does not, so gpt-4o's trade is the right one and mini's is the wrong one — but
this is a *disposition* difference, not a capability gap.

### Gold is a model too — verification rate

Hand-checked the 9 turns where gold and production disagree (the hardest subset
by construction): **gold right on 6, weak or wrong on 3.** Every gold failure was
the same class — naming furniture or a prose fragment instead of a space ("the
chair", "the dinner table", "the hall" for a passage about a table and a hearth).
**Placehood defeats the labeller too.** Treat the absolute numbers as ±10pts and
the direction as sound.

Two findings worth keeping from that read:

- **seq 49** — the player walks out of the War Room, the prose says *"The hall
  outside is quiet"*, and production keeps `War Room`. The durable-stale-cursor
  bug, caught by measurement rather than by a playtest.
- **Neon Divide seq 3** — a terminal readout names `Whisper's Edge` as a third
  party's *last known location*; **both** models put the player there. Textbook
  phantom travel from a mention, and the citation stack refuses it.

### Placehood — five independent confirmations, one day

1. The cursor moved `the hall → the old bench` on a valid (a)(b)(c) citation.
2. `"at the table"` beat `"the royal council chamber"` on passage support.
3. Junk-label rates across four tiers: 18.8% / 8% / 15% / 5.9%, and *every* tier
   emitted at least one.
4. Gold's `place_is_space` flag independently caught production's furniture.
5. The gold labeller itself named furniture on 3 of 9 hard cases.

This is no longer a hypothesis. **It is the load-bearing defect**, it is not
solved by a bigger model, and it is not solved by a longer word list.


---

## Durability, harm reduction, and the first SHIPPED number

### Durability — the actual fix for the four-month class of bug

Accuracy was never the problem. **Durability** was: the cursor was a variable that
could only ever be ASSIGNED, on a turn a move was detected, so one bad or one
MISSED read was permanent. No extractor gets accurate enough to make a
permanently-wrong variable safe.

- **Cursor drift** (`worker/lib/cursor-drift.ts`). A SCENE ANCHOR is computed
  every turn — what this passage says, citation-verified — whether or not
  anything moved. When it contradicts the cursor about the SAME place twice
  running, the cursor re-derives itself with **no move ever detected**, which is
  exactly the case the old design could not represent. Any agreeing turn clears
  the counter, so a stray read cannot accumulate.
- **Party decay.** Membership ended only on an explicit parting phrase, which
  free prose rarely produces. Two consecutive scene breaks without the endpoint
  judge placing them or the prose showing them acting, and they are no longer
  travelling with you. One appearance resets it.

`audit:durability`, 14 cases including the literal War Room turn. 26 audits green.

### Harm reduction — the first hypothesis was wrong

Prediction: mini's 34% harmful-write rate is a *disposition* gap and therefore
promptable. The prompt was made explicit — abstaining is correct and free,
guessing is expensive, furniture is not a place.

| | before | after |
|---|---|---|
| HARMFUL — bad write | 34.0% | **32.0%** |
| SAFE — nothing to say | 2.0% | 4.0% |
| RIGHT | 64.0% | 64.0% |

One turn out of fifty. **Falsified.** In hindsight predictable: the prompt
already said "the room / here / inside are NOT place names" and mini emitted all
three anyway. This is not instruction-following, it is calibration — mini cannot
tell when it does not know. Harm reduction therefore has exactly two levers: move
the seam to a better model (34% → 18%, 3.6× post-stream latency), or let the
gates absorb it.

### The number nobody had ever measured

Every accuracy figure above scores the RAW extractor claim — the *input* to the
gates, not the output. `corpus:shipped` scores the event's own `location_anchor`:
what the map actually ended up holding.

**Overall: 78.4% correct over 74 turns, and the cursor was never unset.**

| | old stack (seq < 44, n=48) | current stack (seq ≥ 44, n=26) |
|---|---|---|
| correct | 85.4% | 69.2% |
| wrong place | 14.6% | 19.2% |
| placehood dispute | 0% | 11.5% |
| no cursor | 0% | 0% |

**Do not read that as a regression.** The two buckets are not comparable: the
current-stack turns are ones deliberately driven with adversarial movement today
(walks to the cellars, the north wall, back to the hall, sleeping, benches),
while the old-stack turns are mostly stationary dialogue. Three of the five
current-stack errors are turns that ran BEFORE the corresponding fix landed
(`the old bench` ×2) or are gold errors (`the hall` labelled `hearth`). A valid
before/after needs the same turns through both stacks, which does not exist yet.

**What IS clean, and it is the important part:** *six of the seven old-stack
errors are the ANCHORING class* — `Split Lamp` held for four turns while the
scene was on the canal bridge, `the green room` held while the scene was on the
loading dock. That is precisely the failure the endpoint judge's `location_at_end`
second namer was built for, and it is the dominant real-world error mode, not a
tail case.

### Where 99% actually stands

Shipped cursor is ~78%. The gap to 99% decomposes as:

1. **Anchoring** (6/7 of observed errors) — addressed by the second namer; needs a
   controlled before/after to confirm.
2. **Placehood** — 11.5% of current-stack turns are a dispute about whether the
   named thing is a place at all. Unsolved by every model tier and by the gold
   labeller. Needs promotion-by-traversal/containment.
3. **Staleness** — addressed by drift repair; no live repair has fired yet, so it
   is verified only by fixtures.

### Placehood — the design, not another list

"Is this string a place" is unanswerable in principle: *the table* is furniture in
a dining room and a location in a world where The Table is a council chamber.
The answerable question is *"is this something the world's map should contain"*,
and it is structural:

> A place is something you can be inside and LEAVE. Furniture is something you
> are AT while remaining inside a place.

Two relational tests, neither lexical:

- **Traversal** — does the prose show the viewpoint entering or leaving it?
- **Containment** — does it contain, or sit inside, something else in the scene?
  (The witness already emits `containment_hint`.) A bench never contains the
  hearth; a hall does.

So placehood becomes **two tiers**: a SCENE ANCHOR (one verified citation,
ephemeral, wrong is harmless) and a MAP NODE (accumulated traversal/containment,
durable). Furniture accrues neither, ever, and "the canal" — absent from
`PLACE_STEMS` — accrues both within a turn or two. This is the same shape as the
carding rule the project already proved for characters in
`unnamed_character_carding`: **recurrence beats naming.**

The anchor half is built. Promotion is the remaining work, and it is the last
hardcoded list that decides fiction.


---

## The controlled before/after

`corpus:location-ab`. Both decision stacks replayed over the SAME 74 turns with
the SAME cached extractor output, each threading its OWN cursor forward in
sequence order — because the failure being measured is a wrong cursor
PERSISTING, which a per-turn comparison cannot see. Extractor calls are made once
and shared, so the delta is the logic, not model sampling.

The decision moved out of the processor into `worker/lib/location-decision.ts`
as a pure function, with `decideLocationLegacy` beside it reproducing the
pre-citation stack. **The processor calls the real one**, so the A/B measures
shipped code rather than a re-implementation that can drift from it.

| | legacy | current |
|---|---|---|
| correct | 62 (83.8%) | **64 (86.5%)** |
| WRONG place | 12 (16.2%) | **7 (9.5%)** |
| placehood dispute | 0 | **3** |
| cursor never set | 0 | 0 |
| moves committed | 7 | **14** |

**Wrong-place turns fall 42% (12 → 7).** The headline only moves +2.7pts because
three of the recovered turns land in the placehood bucket instead.

### What the current stack fixes

`root cellars` held while the scene was in the great hall (seq 43); `the hall`
held for three turns at the north wall (46-48); `the green room` held while the
scene was on the loading dock (seq 16). All anchoring — the class the second
namer was built for. It also **doubles committed moves, 7 → 14**, which is the
intended behaviour: the legacy stack was refusing real moves, not preventing
false ones.

### What it does not fix, and this was predicted wrong

**The City That Watches fails identically in both stacks** — `Split Lamp` held
for four turns (19, 22, 23, 24) while the scene was on the canal bridge. I
claimed the second namer had fixed this class after seeing it work live on one
turn. Replayed properly, with the cursor threaded, **it does not.** One live turn
was not evidence; the sequential replay is.

It also introduces two errors: seq 45 (`north wall` where gold says `hall`) and
seq 17 (`the dock` where gold says `green room`) — the cost of being more willing
to move.

### Reading it honestly

n=74 across three worlds; gold is a model, hand-verified at ~67% on hard cases,
and one of the seven remaining errors (seq 49, gold `hearth`) is a gold error
that penalises both stacks equally. The direction is solid, the magnitude is
±few points, and the residual is now concentrated in exactly two places:

1. **One world's anchoring is still completely unsolved** (4 of 7 errors).
2. **Placehood** — 3 disputes that did not exist before, bought by moving more.

Both were already the named next pieces of work. The A/B did not change the
roadmap; it removed the guesswork about how much each is worth.


---

## Minting is self-justifying — the mechanism under the placehood bug

Investigating why one world's anchoring survived every fix turned up two bugs
and one mechanism that reframes the whole problem.

**Corpus bug (the fourth).** `corpus-freeze` queried `entities` by `kind:
'location'`; the field is `type`, and the name is `canonical_name`, not
`display`. **`knownPlaces` was `[]` for every world in the corpus** — so every
tier run, the gold labelling and the first A/B all ran with the world's map
hidden from them.

**Verifier bug.** A place name can carry its own locative preposition — "under
the bridge", "behind the bar". `excerptSituatesViewpoint` looks for a locative
governor BEFORE the name, so on those it found the preceding noun instead ("the
air under the bridge" → governor "air") and refused a perfect citation. Fixed by
stripping the preposition: the name already asserts the relation.

### The corrected A/B — and the previous number was inflated

| | legacy | current |
|---|---|---|
| correct | 62 (83.8%) | 62 (**83.8%**) |
| WRONG place | 12 (16.2%) | 9 (12.2%) |
| placehood dispute | 0 | 3 |
| moves committed | 7 | **19** |

The earlier 86.5% was an artifact of the empty `knownPlaces`: with the map
hidden, the hygiene gate was accidentally stricter and the current stack made
fewer bad moves. **With the map restored, the two stacks are EQUAL on
correctness.** The current stack converts three wrong-place turns into three
placehood disputes and commits 2.7× the moves. The anchoring fixes are real; they
are exactly offset by placehood errors the extra movement buys.

### The mechanism

```
places minted in Vesperkeep: ["the hall","north wall","root cellars",
                              "the hall","the old bench","root cellars"]

isSafeWitnessLocationCandidate("the old bench")                    → false
isSafeWitnessLocationCandidate("the old bench", { knownPlaces })   → TRUE
```

**A bad place, once minted, becomes a "known place" — and `knownPlaces` is a
short-circuit that means "this is definitely a place."** The gate that would
refuse it is disabled by the graph's own memory of the mistake. One furniture
write on one turn permanently defeats the check for the rest of the run.

This is the durability thesis at the graph layer rather than the cursor layer,
and it is self-reinforcing rather than merely persistent. It also explains why
"the old bench" reappeared in the corrected A/B and not the first one.

Note the same listing shows `"the hall"` and `"root cellars"` minted **twice
each** — the article/singular-plural duplication predicted from the cross-tier
agreement measurement, live in production data.

### What this settles

The two-tier place model is no longer a design preference, it is required:

- a **scene anchor** may be wrong, costs nothing, and mints nothing;
- a **map node** requires accrued traversal/containment evidence, because
  *minting is self-justifying and cannot be undone by a downstream check*.

Until that exists, every location gate in the system has a hole that any single
bad turn can open permanently.


---

## The promotion path — the two-tier map, built

`worker/lib/place-promotion.ts`. A name the cursor lands on is a **scene
anchor**; it becomes a **map node** only once the world has watched it behave
like a place.

| | scene anchor | map node |
|---|---|---|
| evidence | one verified citation | accrued structural relations |
| minted | never | yes |
| in `knownPlaces` | **no** | yes |
| wrong costs | one turn | the rest of the run |

**Promotion signals, none of them lexical:**

1. `authored` — a typed travel destination the player chose in the product, or a
   place the template already knows. Not model output; needs no accrual.
2. `containment` — a validated containment edge. A bench is never described as
   containing a scene or sitting inside a district.
3. `entered_and_left` — the prose showed the viewpoint move INTO it and OUT OF
   it. This is the definition of a place, observed rather than assumed.
4. `recurrent_arrival` — three sightings with at least one arrival. Furniture the
   narration keeps naming never accrues an ENTRY, so it never lands here.

`classifyPlaceRelation` reads the relation from closed grammatical classes only —
entry prepositions (`into`, `onto`, `through`, `to`, `down`, `back`) versus exit
(`out`, `from`, `off`) versus static (`in`, `at`, `on`, `beside`). Static
locatives prove nothing: a bench takes them and so does a hall. It is clause
scoped and viewpoint owned, so *"Bram went down to the cellars"* accrues nothing.

Two rules earned by failing fixtures: the **nearest** preposition governs
(*"step out onto the bridge"* is an entry, not an exit), and `of` is a particle
(*"walk out of the hall"* is an exit, governed by `out`).

`audit:place-promotion` — 20 cases. **27 audits green.**

### Storage and hygiene

`place_candidates`, one row per instance per candidate, unique on
`(instance_id, name_normalized)`. Added to **all three** deletion purge sites —
`deletion.service.ts` has three that drift apart, per the reset-leak history.

`LocationAnchorDoc.entity_id` is now nullable: a provisional anchor names the
setting for the narrator and the cursor and is refused everywhere durable —
`applyLocationFacts` skips it (no node to hang canon on), the context packet
skips ancestry enrichment, memory writes `undefined` rather than null.

### Live

```
seq 72  "I sit down on the old bench by the wall."
        candidate="the silent hall"  promote=false  reason=provisional   ← not minted
seq 73  candidate="the hall"         promote=true   reason=authored
seq 74  candidate="root cellars"     promote=true   reason=authored
```

`the silent hall` is a fresh name variant of a known place. Under the old code it
would have minted a **duplicate map node** and then appeared in `knownPlaces`,
where it would justify itself forever. It accrued as provisional, drove the
cursor for its turn, and wrote nothing.

### Still outstanding

The graph carries pre-existing damage from before the gate:
`["the hall", "north wall", "root cellars", "the hall", "the old bench", "root cellars"]`
— the furniture node and two duplicate pairs. The gate stops new damage; it does
not repair old. `merge:location` exists for exactly this and has not been run.


---

## The graph repair, and the A/B on a clean map

The duplicates were never article variants. The unique index is
`(instance_id, type, world_root_id, parent_id, name_normalized, identity_scope)`
— **`parent_id` is part of the key**, so the same normalized name mints again
under a different parent. On Vesperkeep that produced:

```
"the hall"     seq 2-73   parent=null
"the hall"     seq 56-70  parent=<the hall>     ← a place inside ITSELF
"root cellars" seq 33-74  parent=null
"root cellars" seq 54-67  parent=<the hall>
"the old bench" seq 63    parent=<the hall>     ← furniture
```

`merge:location` could not fix these: it looks a place up by name, and both rows
share a normalized name, so `findOne` returns the same document twice. It now
also accepts an entity `_id`, and `repair:duplicate-places` scans for
same-name groups (keep = earliest seen) plus any self-parented row, dry-run by
default.

Applied to Vesperkeep: 2 merges, **16 event anchors, 11 memories and 22 travel
labels re-pointed**, cursor unchanged. The furniture node was then folded into
its parent hall — the bench really is in the hall, so its 2 event anchors belong
there. Final map: `the hall`, `north wall`, `root cellars`. The other two keeper
worlds had none.

### The A/B on the repaired graph

| | legacy | current |
|---|---|---|
| correct | 62 (83.8%) | **64 (86.5%)** |
| WRONG place | 12 (16.2%) | **7 (9.5%)** |
| placehood dispute | 0 | 3 |
| moves committed | 7 | **17** |

**Wrong-place turns down 42%.** And this closes the loop on three runs that
disagreed:

1. First A/B: 86.5% — but `knownPlaces` was empty from the corpus bug.
2. Corpus fixed: 83.8%, dead level — because the junk `the old bench` node was
   now visible in `knownPlaces` and re-enabled the two bench errors.
3. Graph repaired: **86.5% again, for the right reason.**

The gain was real the whole time and was being cancelled by scar tissue the
promotion gate now prevents. That is the clearest single argument for the
two-tier map: one bad mint, months later, was still eating the benefit of a
correct fix.

### The residual is now one world

Four of the seven remaining errors are **The City That Watches** — `Split Lamp`
held while the scene is on the canal bridge — and both stacks fail them
identically. Of the other three, one (seq 49, gold `hearth`) is a gold error and
one (seq 45, `north wall` vs `hall`) is arguable. The location work has narrowed
to a single world's failure mode, which is where it should go next.


---

## The City world — a feedback loop, not an extractor failure

Four of the seven remaining A/B errors were one world, failing identically in
both stacks. The cause was neither anchoring nor placehood.

### The loop

The narrator is told `CURRENT PLACE: <cursor>` on every turn. When the cursor is
stale and the player writes their own movement, **the narrator follows the cursor
and writes them back where they were** — and that prose then re-confirms the
stale cursor to every post-stream extractor.

```
seq 19  player: "I stop under the bridge and watch the water."
        narrator: "the air under the bridge is heavy with the promise of it"
        cursor: Split Lamp  (unchanged)
seq 20  player: "I think about the girl from the arcade"
        narrator: "He's STILL BEHIND THE BAR"        ← retconned by the cursor
seq 21  narrator: "He doesn't move from behind the bar"
seq 22  player: "We stop on the bridge"
        narrator: bridge prose again
```

The fiction oscillates. **No extractor improvement can fix this**: by the time
the extractors run, the prose has already been written the wrong way. It also
explains why both stacks failed identically — they were reading an incoherent
world faithfully.

### The cause was a missing word in one of two lists that disagree

The pre-stream guard that should have prevented it emits a
`PLAYER MOVEMENT COMMITMENT` directive only when
`extractExplicitPhysicalDestination` fires — which requires the destination to
match `PHYSICAL_DESTINATION_WORD`. That list does **not** contain `bridge`.
`PLACE_STEMS`, in the same file, does.

```
extractExplicitPhysicalDestination("I walk to the canal bridge.")  → null
```

So the narrator was never told, and a world's map sat in a bar for a dozen turns
because two place vocabularies in one file disagree about one word. This is the
standing rule's failure mode in its purest form, and the fix is emphatically not
to add `bridge`.

### `extractStatedPosition` — quote the player, resolve nothing

A first-person clause whose predicate carries a locative or directional
preposition yields **the player's own words**, verbatim, as a narrator directive:

```
"I stop under the bridge and watch the water."  → "under the bridge"
"I walk to the canal bridge."                   → "to the canal bridge"
"I sit by the hearth and say nothing."          → "by the hearth"
"I want to head for the cafe later."            → null   (an intention)
"I look at Soren."                              → null   (a person)
"I nod."                                        → null
```

No vocabulary, nothing minted, nothing validated against a place list. People are
filtered by the caller's cast and by NP shape — a position takes a determiner
("under THE bridge"), a bare capitalised name after a preposition is somebody.

Also landed: the judge's location may now be corroborated by the **player's own
text** as well as the narration. Second-person and sentient worlds rarely write
"you are at X" — The City narrates as *"She's still at the bridge"*, where the
narrating entity is the city itself, so the one locative sentence is owned by a
third person. The judge read it correctly and the verifier refused it.

### Live — the loop is broken

Same four player inputs, replayed on the current stack:

```
seq 34  player: "I stop under the bridge"
        narrator: "The bridge swallows sound... Soren stays on the canal's edge"
seq 35  player: "I think about the girl from the arcade"
        narrator: "She watches you UNDER THE BRIDGE"      ← no retcon
        cursor: the canal
seq 36  cursor: the canal
```

Previously the equivalent turn produced *"He's still behind the bar."*

### A method note

Every fixture appended to `location-evidence-audit.ts` in the previous three
sessions sat **after `process.exit()`** and had never run. Moving the summary to
the end surfaced a real failure immediately: shape B of `excerptSituatesViewpoint`
(the loosest — any first-person clause mentioning the place) accepted *"I want to
head for the cafe later"* as being at the cafe. Fixed with an intent filter on
shape B only; A and C are anchored by grammar already. The other four audits were
checked for the same defect and are clean.
