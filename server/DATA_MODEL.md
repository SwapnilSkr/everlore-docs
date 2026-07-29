# Everlore Server — Data Model

MongoDB collections and how they relate. Canonical names live in `everlore-server/src/models/collections.ts`. Document shapes live in `everlore-server/src/models/*.ts`. Indexes are defined and reconciled at startup in `everlore-server/src/config/mongo-indexes.ts`.

> **IDs:** All primary keys are MongoDB **`ObjectId`** (24-char hex). There are no `usr_` / `tpl_` string prefixes.

For projection lifecycle rules see [WORLD_MODEL.md](./WORLD_MODEL.md) and the memory projection notes. For kinship edge semantics see [KINSHIP_GRAPH.md](../memory/KINSHIP_GRAPH.md). For Ink / entitlements see [BILLING.md](./BILLING.md). Env and deployment: [CONFIGURATION.md](./CONFIGURATION.md), [DEPLOYMENT.md](./DEPLOYMENT.md).

---

## Architecture diagram

```text
users ──owns──► personas
  │
  ├──plays──► world_instances ◄──seeded from── world_templates
  │                 │
  │   ┌─────────────┼──────────────────────────────┐
  │   ▼             ▼                              ▼
  │ events      characters ◄──1:1── entities    story_calendars
  │   │             │                │             timeline_branches
  │   │             └──── entity_edges (meters, narrative, kinship)
  │   │
  │   ├──► memories  (Pinecone mem_{instanceId})
  │   ├──► scene_summaries → chapter_summaries → arc_summaries
  │   │         (Pinecone sum_{instanceId})
  │   ├──► location_stats          (Places tab projection)
  │   ├──► relation_candidates     (player review queue — not canon)
  │   ├──► projection_checkpoints + projection_checkpoint_chunks
  │   ├──► projection_anomalies    (extraction drift log)
  │   └──► signal_ledger           (FP/FN detector metrics)
  │
  ├──► ink_ledger / billing_entitlements / store_purchases
  │
  └──► generation_logs / dead_letter_jobs

Shared / global:
  nsfw_lexicon

Pinecone (per template/instance):
  lore_{templateId}   mem_{instanceId}   sum_{instanceId}
```

Every collection name in `COLLECTIONS` is documented below.

---

## Core gameplay

### `users`

One account per player (password, Google, and/or phone).

| Field | Purpose |
|-------|---------|
| `email`, `phone`, `username`, `password_hash` | Auth identity (email/phone sparse-unique) |
| `google_sub` | Google subject (sparse unique) |
| `providers` | Auth provider labels |
| `tier` | `free` \| `premium` \| `creator` (may be overridden by live entitlement) |
| `preferences` | `nsfw_enabled`, `preferred_model`, `theme`, `narration_length`, `auto_memory_curation`, optional `player_name` / `gender` / `interests` |
| `token_balance` | Legacy field; **Ink balance is derived from `ink_ledger`**, not this column |
| `created_at`, `updated_at` | Lifecycle |

Indexes: sparse unique on `email`, `phone`, `google_sub`; unique on `username`.

---

### `world_templates`

Authored world / character blueprint (draft or published).

| Field | Purpose |
|-------|---------|
| `creator_id`, `title`, `slug` | Ownership + unique public slug |
| `kind` | `world` (RPG) or `character` (lightweight chat); default `world` |
| `is_published`, `is_sentient`, `is_nsfw_capable`, `version` | Publish + capability flags |
| `seed_prompt`, `global_lore` | Premise + lore (lore also embedded to Pinecone) |
| `narrative_style`, `style_notes` | Creator-locked voice |
| `image_url`, `image_prompt` | Cover / avatar |
| `opening_line` | Optional greeting for sentient templates |
| `protagonist` | Locked main persona (`name`, `persona`, `appearance`) |
| `seed_cast[]` | Authored starting cast (copied into each instance as `seed_cast_snapshot`) |
| `base_stats_template`, `flag_definitions`, `scene_tags` | RPG definitions |
| `model_preferences` | Optional per-world narration/summary model overrides |
| `max_context_memories`, `max_lore_results` | Retrieval budgets |

---

### `world_instances`

One playthrough / save for a published template.

| Field | Purpose |
|-------|---------|
| `template_id`, `template_version` | Source template |
| `seed_cast_snapshot` | Immutable-at-start cast copy (rewind never reads a later-edited template) |
| `player_id` | Owner |
| `world_state`, `active_flags` | RPG stats / flags |
| `current_scene` | `{ tag, turn_count, summary_pending }` |
| `narration_pov`, `mode`, `message_length` | Session narration / chat mode / reply length |
| `narrative_style_override`, `narration_tone` | Player voice overrides |
| `focus_character_id` | Optional focused side character |
| `current_time_anchor`, `active_timeline_id`, `default_calendar_id` | Story time cursor |
| `current_location` | `{ entity_id, name, name_normalized }` place cursor |
| `travelling_with[]` | Persistent companions (entity-bounded; survives scene breaks) |
| `manual_relation_assertions` | Player-authored kinship (reapplied after premise on rebuild) |
| `manual_lifecycle_transitions` | Player-confirmed deceased/estranged/… revisions |
| `manual_identity_revisions` | Confirmed rename/merge revisions |
| `persona_id`, `persona_snapshot` | Selected account persona (snapshot avoids drift) |
| `meta` | Totals, archive flag, milestones, fate-seed cursor, `last_continuity_audit` |

See [WORLD_MODEL.md](./WORLD_MODEL.md) for cursor / party / authority rules.

---

### `events` — source of truth

Collection name: `events`. Every turn (main story, calendar tick, travel, intimate, side chat) shares one monotonic `sequence` per instance.

| Field | Purpose |
|-------|---------|
| `instance_id`, `player_id`, `sequence` | Identity (unique `(instance_id, sequence)`) |
| `type` | `narration` \| `calendar_tick` \| `travel` \| `intimate` \| `side_chat` \| … |
| `side_chat` | Private chat ref (`character_id`, entity, name, reachability `mode`, visibility) |
| `data.player_input` / `player_spoken_input` / `player_narration_facts` | Player text |
| `data.world_action` | Structured confirmed action (e.g. travel) |
| `data.ai_response` | Narration prose |
| `data.choices` | Tap-to-play chips (`label`, `kind`, `send`) |
| `data.present_characters` | Canonical names present at end of turn |
| `data.trackable_mentions` | Backend-owned canon-gap mentions (tiered) |
| `data.codex_deltas` | Ledger for exact codex rebuild on rewind |
| `data.location_deltas` / `time_delta` | Authority-tagged place/time projection ledger |
| `data.time_advanced`, `data.travel`, `data.milestone`, `data.fate_thread` | Calendar / travel / seals |
| `data.state_mutations`, `data.flag_mutations` | Game state ops |
| `data.replay_variants` / `selected_replay_index` | Alternate AI responses + per-variant chips/presence |
| `data.model_used`, `tokens_in` / `tokens_out`, `prose_hygiene_issues` | Telemetry |
| `is_user_edited`, `edit_history` | Edit provenance |
| `scene_tag` | Scene block for summary triggers |
| `nsfw_intent` / `nsfw_intent_source` | Sexual-intent momentum for next-turn routing |
| `time_anchor`, `location_anchor` | When / where |

Side-chat events share the sequence counter but are excluded from the main Play feed, scene summaries, and main-visible memory scope.

---

### `memories` — searchable fact atoms

| Field | Purpose |
|-------|---------|
| `text`, `type`, `importance`, `is_nsfw` | Content + ranking |
| `source_event_ids`, `pinecone_id` | Provenance + vector id in `mem_{instanceId}` |
| `is_archived`, `status` | Retrieval gate + projection lifecycle (`active` / `stale` / `superseded` / `archived`) |
| `origin`, `known_by_entity_ids` | Side-chat privacy scope |
| `subjects` / `objects`, `subject_entity_ids` / `object_entity_ids` | Name + graph links |
| `search_terms` | Alternate phrasings (text index + embedding) |
| `time_anchor`, `timeline_id`, `location_anchor` / `location_entity_id` | Filters |
| `emotional_*`, `relationship_delta` | Rich atoms |
| `unresolved_thread`, `resolved_at` | Open promises |
| `updates_memory_ids` / `extends_memory_ids` / `derives_from_memory_ids` | Version graph |
| `superseded_by_event_ids` | Supersession lineage |
| `access_count`, `last_accessed_at` | Decay / ranking |

Text index: `text`, `subjects`, `objects`, `search_terms` (`idx_memories_text_search_v2`).

---

### `characters` — codex cards

Emergent NPC / protagonist cards that become prompt constraints.

| Field | Purpose |
|-------|---------|
| `canonical_name`, `name_normalized`, `aliases` | Identity (unique per instance+normalized name) |
| `role`, `appearance`, `persona` | Sheet |
| `immutable_facts`, `mutable_state` | History vs current status |
| `interaction_hints` | Display-only conversation affordances |
| `disposition_to_player`, `hidden_thought` | Narrative stance |
| `relationship` | Meters: trust / affection / fear / rivalry (0–100) |
| `relationship_state`, `relationship_facts`, `relationship_moments` | Bond summary + evidence journal |
| `entity_id` | 1:1 graph link |
| `is_protagonist` | Locked main persona |
| `first_seen_sequence`, `last_seen_sequence`, `mention_count` | Ranking |

---

### `entities` — graph nodes

Types: `player` \| `protagonist` \| `character` \| `location` \| `faction` \| `item` \| `quest` \| `concept`.

| Field | Purpose |
|-------|---------|
| `canonical_name`, `name_normalized`, `aliases`, `name_tokens` | Identity + bounded mention lookup |
| `character_id` | Link to codex for person types |
| `status` | `stub` \| `anchored_stub` \| `dormant_stub` \| `active` \| `archived` |
| Witness provenance | `source_event_ids`, location ids/sequences, `role_label`, `confidence`, `witness_count` |
| Location-only | `location_facts[]`, `location_state[]`, `parent_id`, `world_root_id`, `place_kind` |

Unique key includes `world_root_id` + `parent_id` so same-named places can coexist across worlds and under different containers. See location / world model docs.

---

### `entity_edges` — graph relationships

Typed directed links with event provenance. Unique on `(instance, source, target, type, label)` so each narrative assertion is its own edge.

| Field | Purpose |
|-------|---------|
| `type` | `trust` / `affection` / `fear` / `rivalry` / `relationship` / `kinship` / … |
| `label` | Free-text for narrative edges; `null` for meters |
| `weight`, `importance`, `status` | Strength / rank / lifecycle |
| `source_event_ids`, `last_event_sequence` | Provenance + rewind clamp |
| Kinship extras | `relation_kind`, `inverse_kind`, `relation_modifier`, `relation_state`, `gender_hint`, `assertion_source`, `confidence`, `since_event_sequence`, `until_event_sequence` |

See [KINSHIP_GRAPH.md](../memory/KINSHIP_GRAPH.md).

---

## Summary tiers (projections)

| Collection | Role |
|------------|------|
| `scene_summaries` | Compressed scene paragraphs; trigger ≈ every 12 turns same `scene_tag`; `event_range`, optional `pinecone_id`, `status` |
| `chapter_summaries` | Rolls up ~8 scene summaries; `chapter_index`, `scene_summary_ids` |
| `arc_summaries` | Rolls up ~4 chapters with plot/relationship framing; `arc_index`, `chapter_summary_ids` |

Embedded in Pinecone `sum_{instanceId}` with deterministic ids `{tier}_{start}_{end}`. Stale summaries are excluded from prompts and rebuilt by the summary queue.

---

## Time & calendar

| Collection | Purpose |
|------------|---------|
| `story_calendars` | Era / month / weekday / moon definitions (`is_default`); template- or instance-scoped |
| `timeline_branches` | Named realities (`timeline_id`, fork metadata, `status`: active / collapsed / alternate / erased) |

`TimeAnchorDoc` (embedded on events/memories/instance cursor): `real_time`, `sequence`, optional `story_calendar`, `event_time_label`, `timeline_id`, `causal_parent_event_ids`, optional subjective times.

---

## Supporting product collections

### `personas`

Reusable account-level player identity (`name`, `gender`, optional `age` / `description` / `other_info`). Instances store a `persona_snapshot` so later persona edits do not rewrite history.

### `nsfw_lexicon`

Routing dictionary for narration model selection. Fields: `term`, `is_phrase`, `category` (anatomy / act / fluid / descriptor / apparel / profanity / other), `weight` (0 = stored only; ≥1 used by classifier), `source`. Worker warms this into memory at startup.

### `generation_logs`

Best-effort per-turn observability (non-blocking). Records NSFW path, models, token counts, prose hygiene issues, `latency_ms`, `ttft_ms`, optional queue / context / end-to-end timings.

### `dead_letter_jobs`

Failed BullMQ jobs persisted for inspection: `queue`, `jobId`, `data`, `error`, `stack`, `failedAt`. Generation final failures write here from the worker.

---

## Projection observability & repair

### `projection_anomalies`

Fire-and-forget inconsistency log (never blocks a turn). Types include `prose_person_untracked`, `kinship_phrase_no_edge`, `choice_ungrounded`, `location_phrase_no_anchor`, `private_fact_leak`, `card_without_prose_anchor`, `orphan_dormant_stub`. Severity: `info` \| `warn` \| `error`. Optional `resolved_at` after repair.

### `signal_ledger`

Per-turn FP/FN measurement for deterministic detectors (`movement`, `time`, `party`, `kinship`, `presence`). One compact row: `player_corrected`, `miss_candidates`, and per-signal tallies (`detected`, `committed`, optional `by_tier` / `source` / `confidence`). Kinship `committed` counts inverse-closed edges (~2× ties) — treat commit% as a trend, not an absolute.

### `projection_checkpoints`

Chunked world-projection snapshots so rewind/replay can restore the latest checkpoint and replay only the suffix. Fields: `sequence`, `kind: 'world_projection'`, `status` (`building` \| `active` \| `failed`), `counts`, optional `instance_state` cursor snapshot, timestamps / `error`. Unique on `(instance_id, sequence)`.

### `projection_checkpoint_chunks`

Payload chunks for a checkpoint: `checkpoint_id`, `kind` (`characters` \| `entities` \| `entity_edges` \| `world_instance`), `index`, `docs[]`.

Scheduled by maintenance cron `projection-checkpoint-scheduler` (see [WORKERS.md](./WORKERS.md)). Service: `projection-checkpoint.service.ts`.

### `location_stats`

Materialized Places tab projection — one row per `(instance_id, entity_id)`. Mirrors display name, containment spine (`parent_id`, `world_root_id`, `place_kind`), `event_count` / `memory_count`, first/last seen sequences. Maintained incrementally on place anchor; backfill available via `locationService.backfillLocationStats`.

### `relation_candidates`

Narrator-originated **review queue**, never canon by itself. Kinds: `kinship`, `identity_rename`, `identity_merge`, `kinship_revision`. Status: `open` \| `accepted` \| `rejected` \| `deferred` \| `superseded`. Player resolve via chronicle API / `relationCandidateService`. Unique on `(source_event_id, character_entity_id, relation)`.

---

## Billing collections

See [BILLING.md](./BILLING.md) for product flows. Shapes in `billing.model.ts`.

### `ink_ledger`

Append-only money/usage record. **Balances are always derived by summing `delta`.** Fields: `player_id`, `delta`, `reason` (`welcome` \| `subscription_cycle` \| `purchase` \| `reserve` \| `settle` \| `release` \| `adjustment`), unique `(player_id, idempotency_key)`, optional `reference`, `created_at`.

### `billing_entitlements`

Current paid access: `tier`, `source` (`google_play` \| `manual`), `product_id`, optional `base_plan_id`, `active`, `expires_at`.

### `store_purchases`

Idempotency boundary around a Google Play purchase token: unique `(provider, purchase_token)`, `status` (`pending_verification` \| `active` \| `consumed` \| `revoked`), product / order metadata.

---

## Pinecone namespaces

| Namespace | Contents |
|-----------|----------|
| `lore_{templateId}` | Template `global_lore` chunks |
| `mem_{instanceId}` | Memory embeddings |
| `sum_{instanceId}` | Scene / chapter / arc summary embeddings |

Memory vector metadata includes `mongo_id`, importance, and side-chat scope fields for fail-closed RAG.

---

## Indexes

Defined in `src/config/mongo-indexes.ts` (`EVERLORE_INDEXES` + `DEPRECATED_INDEXES`), reconciled on Mongo connect. Notable groups:

- Billing uniqueness / wallet scans (`ink_ledger`, entitlements, store tokens)
- Events: sequence uniqueness, scene, timeline, story date, location, side-chat partial
- Memories: archive+importance, text search v2, entity neighborhood, timeline/location/unresolved
- Characters: unique normalized name; rank by mention
- Entities: unique type+root+parent+name; name tokens; parent; character link
- Edges: assertion uniqueness including `label`; provenance multikey
- Checkpoints / chunks, location_stats, relation_candidates, anomalies, signal_ledger
- Users / templates / personas / NSFW lexicon / DLQ `failedAt`

---

## Privacy invariant (data reads)

**Main-story surfaces** must:

1. Filter events: `type: { $ne: 'side_chat' }`
2. Filter memories: protagonist ∈ `known_by_entity_ids` **or** `origin !== 'side_chat'`

Implemented via `memory.service.mainVisibleMemoryScope()` and RAG provider filters.
