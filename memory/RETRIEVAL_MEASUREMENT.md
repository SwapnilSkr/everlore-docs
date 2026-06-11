# Retrieval measurement — the Phase 4 BM25 gate

_Why this doc exists: Phase 4 (broader BM25 over entity names / places / promises
/ event labels) is **deferred, not declined**. The recall gap it would close is a
**scale phenomenon** — it only appears with a long tail of entities/places across
thousands of turns. The ~26-turn dev instance cannot exhibit it, so any eval run
today returns a **false negative**. This is the measurement to run **before**
widening the prompt-recall surface, captured now so the decision is an afternoon's
work once a real instance reaches scale — not a re-litigation._

## What retrieval covers today (the baseline)

Hybrid retrieval in `src/providers/rag.provider.ts` (`queryRag`) fuses five arms
with RRF + an importance boost:

1. **Vector** — Pinecone `mem_<instance>` semantic search.
2. **Keyword (the current "BM25")** — Mongo `$text` index `idx_memories_text_search`
   over **`{ text, subjects, objects }` on the memories collection only**.
3. **Entity-neighborhood** — memories linked BY ID to entities named this turn.
4. **Location** — memories anchored to the current place entity.
5. **Open threads** — always-injected unresolved promises/conflicts.

So exact-name recall already comes from two directions: the text index on the
atom's own words/subjects/objects, AND the id-linked entity/location arms. Phase 4
would add a **new lexical index over the `entities` collection** (names + aliases)
and possibly event labels — i.e. lexical recall for tokens that appear in *no
memory atom yet*.

## The hypothesised gap (what the broader index should win)

Narrow, real, and **scale-born**:

- **Alias-only queries in Echoes search.** The Echoes search box has no entity
  resolution, so a query using an alias that never appears verbatim in any memory
  text (e.g. "Javert" when atoms say the canonical "Inspector Javert") can miss.
- **Freshly-introduced entities/places** that exist in the graph but aren't yet
  the subject of any atom — lexically unreachable until the broad index indexes
  their names.
- **Event-label recall** for a Timeline/Chronicle search feature (NOT the prompt
  path — events aren't retrieved into the prompt; memories/summaries are).

At ~26 turns with a handful of entities, none of these fire. The eval is only
informative once a real instance has the long tail.

## The measurement

**Golden set** (harvest from a REAL high-turn instance, clone-and-discard):
- Queries = actual player turn texts (the live prompt-RAG queries) + real Echoes
  search strings.
- A targeted **slice**: alias-only, place-name, and just-introduced-entity queries
  — exactly where the broad index should help.
- Relevance labels: a strong judge model, or hand-labelled for the slice.

**Metric**: recall@k and MRR of the **fused** retrieval at `k = maxMemoryResults`
(the real prompt budget), **per slice**, comparing:
- current 5-arm stack, vs
- current + a broader-lexical arm (entity names/aliases/places/event labels).

**Decision rule**: widen the index only if it lifts recall@k on the alias/entity/
place slice by a meaningful margin (propose **≥10pp**) **without** regressing
precision on the general slice (RRF can dilute — watch the non-slice recall doesn't
drop). No lift → retire Phase 4 as deferred-by-design (same disposition as cold
archival / revision counters).

**Scale gate (the honest caveat)**: **do NOT run this on the current dev instance.**
~26 turns can't produce the long tail; a green result there proves nothing. Run
when an instance crosses a turn/entity threshold where long-tail entities actually
exist.

**Harness**: read-only, clone-and-discard (the `replay-edit-audit` pattern),
embeddings only (+ optional LLM judge for labels). No mutation of a real save.

## Privacy

The eval must apply the same `allowsKnowledge` gate as `queryRag` (fail closed) so
a private side-chat atom never counts as a retrievable result for a main-narration
query unless the protagonist is a knower. The golden set's "relevant" labels are
scoped the same way.

## Disposition

Phase 4 stays `[ ]` in `CHECKLIST.md` with this doc as its gate. The first slice
when it's pulled in is the **eval harness + baseline measurement**, not the broader
index — build the index only if the measurement clears the decision rule.
