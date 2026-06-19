# World Model — Witness / Entity / Canon

The world model has three tiers of escalating authority. A thing the story
*shows* this turn is a **witness** observation. A thing the graph references
(but hasn't carded yet) is a **stub entity**. A thing the codex has folded into
a canonical card is **canon**. Each tier feeds the next; promotion is the path
between them, never re-minting.

```
  WITNESS (per-turn)          ENTITY-STUB (graph)         CANON (codex card)
  ──────────────────          ───────────────────          ──────────────────
  raw prose                   entity row                   character card
  presence / departures       status: 'stub'               status: 'active'
  location / movement         (no character_id)            character_id ↔ entity
  time / place facts          kinship endpoint             full fold + meters
        │                           ▲                             ▲
        ├─ extractSceneWitness ───►├─ ensureStubEntity ─────►├─ syncCodexEntities
        │                           │  (scene participant)    │  (promote stub)
        └─ extractChoiceMetadata    └─ kinship ensureStub ───►└─ player track endpoint
```

## Why the split

The old pipeline ran one overloaded `extractSceneMetadata` call that produced
all 15 structured fields (choices, stats, presence, location, time, place
facts). A bad/truncated response in the fragile witness half (presence /
location) could corrupt the choice/stat half, and vice versa. Worse, prose
hygiene ran *before* extraction, so canonical-name anchors the model produced
were erased before the extractors could see them — a never-carded "Mara"
mentioned twice got pronoun-substituted, then the codex's grounding guard
blocked her card, then the kinship writer dropped the "Mara is sister" edge
because Mara had no entity id.

The three-tier model fixes all three failure modes:

1. **Witness and choice/stat are separate LLM calls** — a witness outage falls
   back to safe defaults and never touches choices; a choice outage never
   touches presence/location.
2. **Extractors read the RAW prose**, not the hygiene-repaired transcript —
   canonical-name anchors survive for extraction. Hygiene is purely cosmetic
   (shapes only the persisted/displayed prose).
3. **Uncarded scene participants and kinship endpoints get stub entities** —
   the graph references them by id *now*, and the stub promotes to a full
   entity when the codex card mints (same row, never a duplicate).

## Tiers

### Witness (per-turn, low-authority)

A pure observation of what this turn's prose showed. Derived by
`extractSceneWitness` (a focused, low-token LLM call). Fields:
`present_characters`, `characters_departed`, `current_location`,
`containment_hint`, `movement`, `viewpoint_moved`, `time_elapsed`,
`location_state_changes`, `location_permanent_facts`. Falls back to safe
defaults (empty presence, unchanged location) on failure. Isolated from the
choice/stat half so a witness outage never corrupts choices.

### Entity-Stub (graph, high-recall / low-authority)

A lightweight entity row with `status: 'stub'` and no `character_id`. Created
by `entityGraphService.ensureStubEntity` for any witnessed-but-uncarded person
(scene participants via `ensureSceneParticipantStubs`, kinship endpoints via
the kinship writer's `ensureStub` fallback). Stubs let the graph reference a
person by id *before* a codex card exists — so a "Mara is the player's sister"
tie is captured against Mara's stub now, not dropped. The moment a codex card
mints for the name, `syncCodexEntities` promotes the stub to `status: 'active'`
and links `character_id` (the same row; the unique index made them match). Stubs
are **not** injected as codex canon and **not** shown in the Bonds ledger (that
reads cards) — they are a recall scaffold, not a presentation tier.

### Canon (codex card, highest authority)

A full `CharacterProfileDoc` folded from ledgered deltas by the codex service's
`foldDelta`. A card is canonical, deduped, and exactly rebuildable from the
event ledger (`codex_deltas`). The `mention_count` is a continuous ranking
weight for prompt injection, **not** a tier boundary — a card is minted at the
first grounded, non-blocked appearance with `mention_count = 1` and is
immediately a full card.

## Extraction pipeline ordering (per turn)

| # | Step | File | Prose source |
|---|------|------|--------------|
| 1 | Stream prose tokens to player | `generation.processor.ts` | raw |
| 2 | `generation_stream_end` publish | `generation.processor.ts` | raw |
| 3 | **Prose hygiene** (cosmetic repair) | `prose-hygiene.ts` | raw → hygiened |
| 4 | `finalNarrative` = hygiened prose (persisted/displayed) | `generation.processor.ts` | hygiened |
| 5 | **`extractSceneMetadata`** (witness + choice/stat in parallel) | `metadata-extractor.ts` | **raw** |
| 6 | `groundChoices` | `choice-grounding.ts` | **raw** |
| 7 | Insert event (`data.ai_response` = hygiened) | `generation.processor.ts` | hygiened |
| 8 | **`ensureSceneParticipantStubs`** (witness → stub) | `entity-graph.service.ts` | — |
| 9 | **`extractCharacterCodexDeltas`** | `character-codex-extractor.ts` | **raw** |
| 10 | Sentient-player self-card guard | `generation.processor.ts` | — |
| 11 | `characterCodexService.applyDeltas` (fold + persist) | `character-codex.service.ts` | — |
| 12 | Ledger deltas on event (`data.codex_deltas`) | `generation.processor.ts` | — |
| 13 | **`syncCodexEntities`** (stub → active promotion) | `entity-graph.service.ts` | — |
| 14 | `syncRelationshipEdges` (meter edges) | `entity-graph.service.ts` | — |
| 15 | **`applyRelationAssertions`** (kinship edges, `ensureStub` fallback) | `kinship-graph.service.ts` | **raw** |

**Key invariant**: steps 5, 6, 9, and 15 read the **raw** prose (canonical-name
anchors survive); step 7 persists the **hygiened** prose (what the player sees).

## REST contracts

### `POST /chronicle/track/:instanceId` — promote / correct

Player-driven surface that repairs a projection miss ("Track this character" /
"This person is my sister"). Writes an EVENT-DERIVED projection, not a free-form
card mutation:

1. Mints/updates the codex card via the same `applyDeltas` fold the turn
   pipeline uses (canonical, deduped, replayable).
2. Promotes the matching scene-participant / kinship stub entity to active and
   links `character_id` (`syncCodexEntities`).
3. Optionally writes a typed kinship edge (`ensureStub` creates a stub for a
   still-uncarded endpoint so the tie is captured immediately).
4. Ledgers the synthetic delta on the most recent main-story event's
   `codex_deltas` so a rewind replays the card. (Kinship edges are not
   ledger-replayed today; a rewind past this point may drop the edge.)

A player correction is treated as `narrator`-level authority (full confidence
0.9), not a `character_claim` (0.5) — the player is the author of their world's
relationships, not a character making an in-fiction claim that may be a lie.
The endpoint publishes a `character_codex_updated` WS frame after tracking so
every tab/device reconciles to the authoritative full list (the caller's
optimistic splice is a hint, not truth).

**Request body:**
```json
{
  "name": "Mara",                          // required, 1-120 chars
  "role": "the twin sister",               // optional
  "appearance": "string",                  // optional
  "persona": "string",                     // optional
  "relation_kind": "sibling_of",           // optional; one of the closed RelationKind enum
  "relation_label": "twin sister",         // optional; world-native term
  "relation_to": "player"                  // optional; defaults to "player"
}
```

**Response:** `{ character: CharacterProfile, relation_asserted: { kind, label, to } | null }`

Sentient worlds refuse to card the player persona (mirrors the generation-time
self-card guard). The endpoint is idempotent: tracking an already-carded name
updates the card rather than minting a duplicate.

## WebSocket frames

No new WS frame types were introduced. The existing `character_codex_updated`
frame (published after `applyDeltas`) reconciles the client's codex list when a
track request lands server-side, so the client's optimistic splice is
authoritatively reconciled. The `generation_complete` frame already carries
`present_characters`; the client-side miss audit (`detectVisibleGaps`) diffs the
latest turn's prose against those + the codex to surface tappable un-carded
names without a round-trip.

## Audits

### `bun run audit:presence-codex-gap` — pure-function gap detector

The "visible in prose but absent from presence/codex/stubs" check. A name
visible in the raw prose that is NOT in the turn's `present_characters`, NOT a
codex card, and NOT a stub entity is a GAP. Passer-by descriptors ("the
merchant", "an old woman") are filtered so the audit flags real missed people,
not walk-on nouns. Family-role words (father, sister, butler, king, …) are
candidates so a present-but-uncarded role NPC the premise introduced surfaces as
a gap the player can promote. Child/self-facing labels (son, daughter, child)
are excluded so the player is never flagged as a missed "Son". Mirrored
client-side by `PlayCubit.detectVisibleGaps()`.

### `bun run audit:kinship` — kinship hygiene + choice grounding

Existing audit, unchanged. Covers ontology inverses, surface-term mapping,
figurative-kinship detection, Stage-1 hygiene (inverse closure, kind repair,
1-hop sibling→child inference), and choice grounding (graph labels, supernatural
reification gating).

## Guard summary (character codex)

| Guard | What it blocks | Where |
|-------|----------------|-------|
| A1 empty name | blank delta | `character-codex-extractor.ts:151` |
| A2 self-relation | `from == to` | `character-codex-extractor.ts:135` |
| A3 ambiguous correction | both names already canonical | `character-codex-extractor.ts:225` |
| A4 bare descriptor | "the merchant", "an old woman" | `character-codex-extractor.ts:88` |
| A5 absent player-relative | "my sister is Mara" off-screen | `character-codex-extractor.ts:41` |
| A6 ungrounded new card | name not literally in prose | `character-codex-extractor.ts:432` |
| A7 cap of 6 deltas/turn | over-extraction | `character-codex-extractor.ts:445` |
| A8 sentient-player self | persona self-intro | `generation.processor.ts:1045` |
| A9 GM protagonist meters | self-meters nonsense | `generation.processor.ts:1117` |

Role-named family NPCs (Father, Sister, Butler, King) are **not** in A4's
blocklist and are explicitly whitelisted in the presence filter
(`trackableFamilyLabels`), so they seed codex extraction as real scene
participants. With the witness/entity/canon split, the raw-prose anchors they
need to pass A6 now survive hygiene, and any that still slip through become
stubs the player can promote via the track surface.