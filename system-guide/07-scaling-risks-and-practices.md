# Scaling, Best Practices & Risks

What stays healthy at 10,000+ turns vs what could hurt you.

---

## What scales well (by design)

| Concern | Why it's OK |
|---------|-------------|
| **Prompt size per turn** | Hard-capped sections + token budgets + recents floor |
| **AI cost per turn** | ~constant LLM calls regardless of history depth |
| **Memory search** | Pinecone top-K + indexed Mongo filters — not full table scan |
| **Rewind at deep history** | Projected event load once; codex rebuild in memory bulk write |
| **Dedup job cost** | Fixed: fetches stored vectors instead of re-embedding all |
| **Summary tiers** | Old turns represented by compressed paragraphs, not raw prose |
| **Client memory** | App keeps 50 live echoes; Chronicle paginates |

---

## What grows with playtime (expected)

| Thing | Growth | Mitigation already in place |
|-------|--------|----------------------------|
| Events collection | Linear with turns | Summaries bridge prompt; cold archive deferred |
| Memories collection | Linear (but dedup merges) | Importance decay archives stale; dedup weekly |
| Pinecone vectors | Linear with memories + summaries | Delete on rewind/archive |
| Entities / edges | Sublinear (dedup by name) | Orphan prune + rewind repair |
| Character codex facts | Bounded per card | Async compaction at 24+ facts |
| Continuity audit | Per instance | Daily, capped 1000 instances/run |

---

## Known risks & bottlenecks

### 1. Dedup is O(n²) in memory count

**Where:** `maintenance.processor.ts` → `dedup_memories`

Compares every memory pair via cosine similarity. Fine for hundreds; slow/expensive at thousands per instance.

**Mitigation today:** Only runs weekly; instances need 20+ memories to trigger.

**Future fix:** Clustering or LSH instead of all-pairs.

### 2. Two embeddings per turn

**Where:** `context-packet.service.ts` calls `queryRag` and `querySummaries` in parallel — each embeds the query.

**Impact:** Small extra cost/latency every turn.

**Future fix:** Share one embedding between both calls.

### 3. Identity scan of full cast each turn

**Where:** `entityGraphService.findMentionedCharacterCards` / codex mention matching

Loads lightweight name list for all characters. Cheap at dozens/hundreds; noticeable at very large casts.

### 4. applyDeltas loads all character cards

**Where:** `character-codex.service.ts`

Needed for alias resolution. Index-backed; acceptable for realistic party sizes.

### 5. Count caps vs token caps (mostly fixed)

Per-section token budgets shipped, but pathological single items (one huge memory text) could still eat space within a section's budget.

**Mitigation:** `withinTokenBudget` keeps at least one item; ranker drops tail first.

### 6. LLM extraction quality (not a scale bug, a correctness bug)

Travel detection, location facts, side-chat scoping, metadata extraction all depend on model following schema. Code paths are tested; **live-turn verification** on real generations is still recommended.

### 7. Mongo storage cost (long horizon)

Full events retained forever by design. No cold tier yet.

**When it matters:** Very long instances at scale — monitor collection sizes before building archival.

### 8. scene_summaries index mismatch (minor)

Queried on `event_range.end_sequence`; index on `start_sequence`. Instance-scoped — low impact until huge summary counts.

### 9. Side-chat privacy depends on discipline

Any new `events().find` or memory read for main surfaces **must** apply exclusion filters. Missing one filter = leak risk.

**Pattern to copy:** grep existing sites in `HANDOFF.md` privacy section.

### 10. Free Pinecone / Render spin-down

Infra note: ephemeral filesystem on Render; free tier spin-down. Workers and Pinecone must reconnect; session in Redis may stale — app uses `load_instance` after rewind.

---

## Best practices summary

```text
┌────────────────────────────────────────────────────────┐
│ DO                          │ DON'T                    │
├─────────────────────────────┼──────────────────────────┤
│ Treat events as truth       │ Send full transcript to AI│
│ Rebuild projections on edit │ Mutate memories silently  │
│ Bound every prompt section  │ Unbounded find() on hot   │
│ Fail-closed on secrets      │ Trust model for privacy   │
│ Async extract/embed         │ Block turn on memory job  │
│ Ledger codex on events      │ Re-LLM codex on rewind    │
│ Test rewind-audit on changes│ Ship graph changes blind  │
└────────────────────────────────────────────────────────┘
```

---

## Operational monitoring

| Signal | Where |
|--------|-------|
| Drift warnings | Logs `continuity.drift`; `meta.last_continuity_audit` on instance |
| Admin unhealthy worlds | `GET /admin/instances/continuity-audits?status=unhealthy` |
| Stuck summaries | `repair_scene_summaries` every 15 min |
| Dead letter jobs | `deadLetterJobs` collection + WS `generation_failed` |
| Generation logs | `generation-log.model.ts` (observability) |

---

## Capacity rough guide (order of magnitude)

These are not hard limits — rough comfort zones with current architecture:

| Metric | Comfortable | Watch zone |
|--------|-------------|------------|
| Turns per instance | 10k–20k+ | Prompt quality depends on summary/RAG quality, not count |
| Memories per instance | < ~2,000 | Dedup O(n²) hurts beyond this |
| Characters per instance | < ~200 | Mention scan + applyDeltas linear |
| Side chats | Unlimited events | Same sequence space as main story |
| Concurrent generations | 3 workers | Queue backlog under load |

---

## What "scale" means for Everlore

**Player experience scale:** Can I play for months and still feel remembered? → **Yes**, architecturally.

**Ops scale:** Can we run 10k instances without melting? → **Mostly**, with monitoring on dedup, Mongo size, and worker queue depth.

**Intelligence scale:** Does the world reason like a full knowledge graph? → **Partially** — entity graph exists; memory-version links and richer lexical index are the next leverage points.
