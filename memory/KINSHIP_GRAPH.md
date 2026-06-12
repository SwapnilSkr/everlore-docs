# Kinship Graph — typed, genre-agnostic relationships with two-stage hygiene

_Implementation spec. The CHARACTER analog of [`LOCATION_GRAPH.md`](./LOCATION_GRAPH.md):
where that doc gives places a duplication-proof, nested identity, this gives the
relationships between people a typed, queryable identity. Read LOCATION_GRAPH first
for the shared patterns (entities + `entity_edges`, provenance pruning, deterministic
closure, world-agnostic design, gated rollout) — this reuses all of them._

## Why

Today the system models player→character **sentiment** (trust/affection/fear/rivalry
meters) and free-form narrative edges ("betrayed", "forgave"), plus location
containment. It does **not** model **typed kinship**: "Mara is the player's sister"
exists only as free TEXT in a card's `role` field. Two failures follow:

1. **Fabricated relations in choices** — the choice extractor invents a relative the
   cast lacks ("Encourage my brother" when the player has only a sister). The current
   guard ([`choice-grounding.ts`](../../everlore-server/worker/lib/choice-grounding.ts))
   scans `role` TEXT for kin words — a heuristic, not knowledge. (See
   [memory] choice-grounding-guard.)
2. **Dangling figurative reference** — "the golden child" points at the sister, but
   nothing maps the epithet to her identity; the choice is ungrounded.

The graph turns "scan role text and hope" into a deterministic question:
`relativesOf(protagonist) → { sibling_of: [Mara], ... }`.

## The core idea that makes it work for ALL worlds (LOCKED)

A closed human enum (`brother|sister|mother…`) breaks on hive-queens, AI forks,
clone-lines, packs, liege-houses. So a relation has **two layers**:

| Layer | Cardinality | Who reads it | Example |
| --- | --- | --- | --- |
| **structural kind** | CLOSED (~8) | the ENGINE reasons over this | `sibling_of` |
| **label** | OPEN free text | players + UI see this; resolver disambiguates | `"clone-sister"` |

The extractor maps **any** world-native term onto one structural kind while keeping
the native label. Code only ever reasons over the closed kinds; the closed set never
needs to grow for a new genre.

### The closed structural ontology

| kind | inverse | symmetry | absorbs (examples) |
| --- | --- | --- | --- |
| `parent_of` | `child_of` | asymmetric | mother, father, sire, dam, creator-parent |
| `child_of` | `parent_of` | asymmetric | son, daughter, heir, scion |
| `sibling_of` | `sibling_of` | **symmetric** | brother, sister, twin, clone-sibling, shield-brother, broodmate |
| `partner_of` | `partner_of` | **symmetric** | spouse, husband, wife, mate, bondmate, betrothed |
| `progenitor_of` | `descendant_of` | asymmetric | maker, creator, clone-source, time-origin, hive-queen→drone |
| `descendant_of` | `progenitor_of` | asymmetric | creation, clone, fork, drone |
| `superior_of` | `subordinate_of` | asymmetric | liege, master, commander, owner |
| `subordinate_of` | `superior_of` | asymmetric | vassal, servant, ward, apprentice, familiar |
| `kin_of` | `kin_of` | **symmetric** | generic "family"/"relative" when no finer kind fits |
| `bonded_of` | `bonded_of` | **symmetric** | sworn-friend, blood-oath, packmate (peer, non-family) |

`gender` is a HINT on the label/card, never part of the kind ("sister" = `sibling_of`
+ gender F). This keeps a trans/agender/non-binary/non-human cast representable.

## Data model

Kinship edges are `entity_edges` (already directed, provenance-tracked, status-aware,
rewind-pruned). We extend [`EntityEdgeDoc`](../../everlore-server/src/models/entity-edge.model.ts):

```ts
type: 'kinship'                         // distinguishes from meter/narrative edges
relation_kind: 'sibling_of'             // CLOSED structural kind (engine reasons here)
label?: 'twin sister'                   // OPEN world-native term (already exists on the model)
inverse_kind?: 'sibling_of'             // the kind written on the auto-closed reverse edge
gender_hint?: 'm' | 'f' | 'n' | null    // from the label, for label↔card consistency
assertion_source: 'narrator' | 'character_claim' | 'inferred' | 'seed'
confidence: number                      // 0-1; ranks competing assertions
status: 'active' | 'ended' | 'retconned' | 'stale' | 'archived'   // extend existing enum
since_event_sequence?: number           // temporal validity (marriage, reveal)
until_event_sequence?: number | null    // set when a relation ENDS (divorce/death/retcon)
// existing: source_entity_id, target_entity_id, source_event_ids, last_event_sequence…
```

Edges connect **resolved entity IDs**, not strings — riding on the codex's existing
`resolved_name` identity resolution. Unique identity = `(instance, source, target,
relation_kind)`; a new assertion on the same triple updates it (not a duplicate).

## Per-turn timeline (TTFT-safe)

```
PRE-STREAM   build context packet → READ graph as it stood after prior turns (cheap, no LLM)
░ STREAM ░   first token → player. Nothing kinship runs here.
POST-STREAM  (turn tail, off TTFT — same slot as codex extraction)
  0. LLM extraction    relation_assertions emitted alongside codex deltas (one existing call)
  1. STAGE 1 hygiene   DETERMINISTIC, no LLM — closure, metaphor-strip, consistency, contradiction
  2. STAGE 2 resolver  MICRO-LLM, ONLY for unresolved person-epithets ("golden child") — usually no-op
  3. write edges       visible to the NEXT turn; choice guard's prose-fallback covers the same-turn gap
```

**Reads** (choice guard, presence, prompt context) are deterministic, no LLM, cheap.
**Writes** (extract → hygiene → resolve → persist) are all post-stream. The
deterministic/LLM split is *within* the tail, NOT pre vs post.

## Stage 1 — deterministic hygiene (no LLM)

Pure functions over the assertion list + roster. Mirrors `prose-hygiene`'s detector.

- **Closure**: assert `parent_of(A→B)` ⇒ also write inverse `child_of(B→A)`; symmetric
  kinds (`sibling_of`, `partner_of`, `kin_of`, `bonded_of`) write both directions.
- **Metaphor/simile strip**: drop `like a brother`, `as a father to me`, `brotherly`,
  `practically family`, `almost a sister` — figurative, not literal kin.
- **Kind↔label & gender consistency**: label "sister" but kind `parent_of`, or label
  "sister" on a male-gendered card → flag/repair to the label-implied kind/gender.
- **Contradiction & cardinality**: two active `parent_of` of the same gender onto one
  child, or `sibling_of`+`partner_of` on one pair, or N active mothers → keep as
  COMPETING ranked by `confidence` then recency; never force-merge.
- **Self-loops, exact dup, edges to unresolved entities** → drop / quarantine.
- **1-hop inference (bounded, tagged)**: A `sibling_of` B ∧ A `child_of` P ⇒ B
  `child_of` P (`assertion_source: 'inferred'`, lower confidence). Capped at 1 hop —
  half/step/adoptive families make deeper inference unsafe.

## Stage 2 — epithet resolver (micro-LLM, only on residue)

Stage 1 resolves the EASY coreferences deterministically (lexical/alias/role overlap:
"the sister"→Sister card; unique structural slot in scope). What remains is genuine
**semantic coreference** — "the golden child" has no lexical hook and two children
exist. Exactly as prose-hygiene escalates its residue to a targeted `callLLM`, we
escalate ONLY the unresolved person-epithets:

```
resolveEpithets({ epithets:["the golden child"], roster, sceneText })
  → [{ epithet, resolved_card_id | null, confidence }]
```

One cheap call, post-stream, off TTFT, fires **only when** Stage 1 left an unresolved
epithet (most turns: never). Maps "golden child" → Mara's card so the choice grounds
to an identity instead of dangling. `null` = leave ungrounded (safe — the narrator
re-interprets the free-text `send` anyway).

## Read path

- `relativesOf(instanceId, selfEntityId)` → `{ [kind]: { entityId, label, name }[] }`.
  Self anchor = the is_protagonist card's entity (GM worlds) or the player entity
  (sentient). One indexed read on `(instance, source_entity_id, relation_kind)`.
- `kinSummary(...)` → denormalized `{ kindsPresent: Set, labelsByKind }` cached on the
  protagonist card for the hot path (choice guard) — zero LLM, negligible latency.

### Choice guard, repointed

`groundChoices` stops scanning `role` text. It maps each choice's surface kin word →
structural kind (genre-aware lexicon) and asks the kin summary whether the protagonist
has an edge of that kind. Absent in BOTH the graph AND this turn's grounded prose ⇒
fabricated ⇒ drop. The prose fallback (already shipped) covers the same-turn lag for a
relative introduced this very turn.

## Hard edges (honest)

| Risk | Handled by |
| --- | --- |
| Metaphor leakage ("like a brother") | Stage 1 simile strip + `character_claim`/confidence |
| Inference explosion (half/step/adopt) | cap at 1 hop, tag `inferred`, low confidence |
| Contradiction (two fathers / reveal) | competing assertions ranked by confidence+recency; reveal = higher-confidence supersede, `until_event` on the old |
| Unreliable narrator (a lie) | `assertion_source: character_claim` stays soft until corroborated |
| Exotic kin (clones/hives/AI forks) | structural catch-alls `progenitor/descendant/kin/bonded_of`; flavor in label |
| Figurative coreference ("golden child") | Stage 2 micro-LLM resolver |
| Identity merge ripple | `merge:character` re-homes edges (extend existing tool) |
| Rewind | `source_event_ids` pruning already exact on `entity_edges` |

## Rollout (gated, shadow-first)

1. **Schema + ontology + extraction + Stage 1 + write path** — graph is SHADOW-written
   on the turn tail; no behavior change. Audit: `bun run audit:kinship`.
2. **Repoint** `groundChoices` + presence to the kin summary (flag `KINSHIP_GRAPH_READS`,
   default on, prose fallback so it cannot regress).
3. **Stage 2 resolver** + temporal reveals + 1-hop inference.
4. **Seed from world creation** (parse premise/character sheets → `seed` edges so turn 1
   isn't empty) + a "Bonds/Relations" UI surface.

## Status

- **This pass implements 1 + 2 + 3** (schema, ontology, extraction, both hygiene stages,
  write+read paths, choice-guard repoint, audit). Phase 4 (creation-seeding, UI) is a
  marked follow-up.
- Files: `src/utils/kinship-ontology.ts`, `worker/lib/kinship-hygiene.ts`,
  `worker/lib/kinship-epithet-resolver.ts`, `src/services/kinship-graph.service.ts`,
  extraction in `character-codex-extractor.ts` + `character-codex.service.ts` (delta
  type), wiring in `generation.processor.ts`, repoint in `choice-grounding.ts`,
  `scripts/audit-kinship.ts`.
