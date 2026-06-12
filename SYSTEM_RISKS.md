# Everlore — System Risk Map (canonical)

_Last updated 2026-06-12. This is the honest, code-grounded map of where an actual
open-world story will break. It supersedes the per-run playtest noise (now in
`playtest/archive/`). Keep it short, keep it true; update it when a risk closes or a new
structural one appears._

## The one principle everything below follows

The system splits work into two layers:

- **Structural layer (deterministic — regex / word-lists / graph rules).** Answers *closed*
  questions: "is this a name? a place? a bare descriptor? is someone present?" Finite answer
  sets, debuggable, and the worst failure is a **miss** (under-fires), not corruption.
  This layer is **solid** and should stay deterministic.
- **Semantic layer (the LLM extractors + RAG).** Answers *open* questions: "what's worth
  remembering, whose feeling is this, is this a correction, is this explicit intent?"
  Infinite phrasings; failure here **corrupts** the story.

**The rule:** use determinism for *shape*, the model for *meaning*. Every real risk below is
either (a) meaning leaking into the deterministic layer, or (b) the model emitting
*inconsistent* output the deterministic layer can't repair. **You cannot regex your way out
of the semantic risks** — adding more words/patterns just moves the next failure.

Severity: 🔴 breaks stories often · 🟠 breaks under specific play · 🟡 annoyance/cosmetic.

---

## A. Memory recall — the highest-leverage risk 🔴

**Persistence ≠ recall.** Memories persist forever in Mongo, but only enter the prompt if
they hit the RAG query (vector + keyword + entity-neighborhood + open-threads, top-K ~25
memories / ~10 lore — `context-packet.service.ts` §2, `queryRag`). Consequences:

1. **Differently-worded facts don't surface.** A memory phrased unlike the current turn can
   simply not be retrieved → "the AI forgot," even though the fact is right there. This is
   the fundamental RAG ceiling for an open world.
2. **No re-ranking** (explicitly deferred — see `backend_optimization`). Top-K by raw vector
   similarity means a trivially-similar memory can crowd out a crucial but differently-worded
   one. *This is the single biggest fix available.*
3. **Lossy summaries.** Long-horizon recall leans on scene/chapter/arc summaries; anything
   they didn't capture is reachable only by a raw-atom vector hit before it ages out of the
   recent window.

**Right fix (not regex):** a re-ranking pass over the retrieved pool (cross-encoder or a
cheap LLM rank), + importance/recency blending. Spec separately.

## B. Contradictory memories co-exist 🔴

Supersession only retires the old fact when the **correction detector** fires
(`memory.processor.ts` correction path + `memory-supersession.service`). A fact that changes
*without* a detected correction leaves **both** versions `active` and recallable → the AI
cites stale and current facts together. Audits pass (both atoms are individually valid).

**Right fix:** broaden correction detection beyond explicit "X not Y" phrasing — and, related,
make the extractor emit **one canonical correction shape** (see E).

## C. B1 identity attribution poisons recall, not just cards 🔴

The privacy/visibility gate keys on `known_by` entities (`mainVisibleMemoryScope`,
`memory.service.ts`). When a player fact is mis-attributed to the AI character (the B1 class),
the memory can land under the wrong owner → hidden when it should show, or shown when it
should be private. So B1 is **not cosmetic** — it corrupts *what gets recalled and to whom*.
Batch-4 added prompt-layer guards (first-person-sensation → player; anti-name-fabrication);
this is a **semantic** fix and needs live confirmation, not an audit.

## D. present_characters drift 🟠

Model lists who's present each turn; server **carries forward** prior-present minus
`characters_departed`; a scene-break (move / time-skip / place-entity change) resets it
(`generation.processor.ts` ~495–535).

- **Lingering ghosts:** the *only* removal path is the model explicitly marking someone
  `departed`. People usually just stop being mentioned → they stay "present" for the rest of
  the scene (run-d "Mira present while trapped", "Alex still present").
- **Presence isn't canonicalized server-side** (unlike the codex): matched by lowercased
  string equality, so alias drift ("the captain" → "Bram") can drop or double-count.
- **Binary scene-break:** any place-entity change wipes everyone; a real move resolving to the
  *same* node won't reset.

**Right fix:** canonicalize presence names through the entity registry (structural, safe), and
consider a soft decay for un-mentioned characters instead of all-or-nothing carry-forward.

## E. Extractor emits inconsistent shapes — the unrepairable seam 🔴

The N3 sister bug is the archetype: the Mara→Mira correction was written in **three different
`name`/`resolved_name` shapes** across turns, so the ledger replay (`rebuildCodexFromLedger`)
can't converge — no deterministic rule fixes it, because the *input* is contradictory. Fresh
instances pass (R2 closed in run d); corrupted ledgers can't be replayed clean.

**Right fix:** constrain the codex extractor to emit corrections in **one** shape
(`resolved_name` = the existing canonical, `name` = the new corrected proper name), validated
on the way in. Then replay is deterministic. This is the second-highest-leverage fix.

## F. Location graph — strongest subsystem, real edges remain 🟠

The cartographer (`entity-graph.service.ts` `placeLocation` / `resolveLocationAnchor`) is well
designed (space ⟂ time, world-roots, area-scoped dedup, lazy depth, vague guards). Limits:

- **Follows the model's witness hints** (`current_location`, `containment_hint`, `movement`,
  `viewpoint_moved`). Deterministic backstops catch *some* misses; the cursor ultimately
  trusts the model.
- **`world_shift` is model-triggered** — a real realm jump the model doesn't flag is hung
  under the wrong root. No deterministic "different world" detector.
- **Wrong parent is sticky:** re-parenting fills an *unknown* parent only; it never corrects a
  wrong one. An early mis-parent persists.
- **Dedup under-merges by design** (safer than over-merge). Batch-4 fixed memory-mention
  fragmentation; cross-area same-names and sub-threshold fuzzy cases still split. Merging is
  manual (`merge:location`).
- **Cursor lag** (run-d seq-8): vague-only narration ("outside") can't move the cursor —
  defensible, left as accepted ambiguity (forcing it re-opens phantom travel).

## G. NSFW routing — word-list can't catch intent 🟠

The lexicon (`nsfw_lexicon`, 500 terms / 202 routable) is a *hardcore-vocabulary* list. It
catches explicit **words**, never explicit **intent** in clean language ("take me", "I give
myself to you"). The run-d miss scored 1 lexicon word vs threshold 3. Batch-4 added a
code-level ambiguous-pattern tier (+1) — a **stopgap on the same treadmill**.

**Right fix:** for borderline scores (1–2), defer to an intent signal that already exists —
chat mode (Ardent forces explicit), account opt-in, or a tiny classifier call — instead of
growing the word list forever.

---

## Fix priority (by leverage, not effort)

1. **A — memory recall re-ranking.** Biggest payoff for "the AI remembers the right thing."
2. **E — one canonical correction shape from the extractor.** Makes replay/supersession sound.
3. **C — B1 attribution, verified live.** Unblocks correct recall scoping.
4. **B — broaden correction detection** (depends on E).
5. **D — canonicalize presence names** (structural, low-risk).
6. **F/G — accept as known edges**; touch only with care (phantom-travel / romance-false-positive regressions live here).

## What is genuinely solid (don't regress)

Deterministic structural guards: vague-location & bare-descriptor filters, article-stripped
location dedup, possessive-room naming, name-appears-in-text grounding, the world-root/area
graph model, side-chat privacy projection, rewind ledger replay *when the ledger is
consistent*, and the full audit suite (`audit:*`). These are the load-bearing walls.
