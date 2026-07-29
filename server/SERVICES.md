# Everlore Server — Service Layer

How the API is organized: **routes → controllers → services → workers**.

See also: [DATA_MODEL.md](./DATA_MODEL.md), [WORKERS.md](./WORKERS.md), [WORLD_MODEL.md](./WORLD_MODEL.md), [KINSHIP_GRAPH.md](../memory/KINSHIP_GRAPH.md), [BILLING.md](./BILLING.md), [CONFIGURATION.md](./CONFIGURATION.md), [API.md](./API.md).

---

## Request flow

```text
HTTP / WebSocket
    → routes/*.routes.ts          (validation, auth plugins)
    → controllers/*.controller.ts (orchestration)
    → services/*.service.ts       (domain logic, Mongo/Redis/Pinecone)
    → queues (async)              → worker/processors/*.ts
```

WebSocket play: `ws.routes.ts` → `play-ws.controller.ts` → `play-ws.service.ts` → `generation.service.ts` (enqueue only) + `billing.service.ts` (reserve Ink before dispatch).

---

## Controllers ↔ routes

| Controller | Route prefix | Responsibility |
|------------|--------------|----------------|
| `auth.controller.ts` | `/auth/*` | Register, login, Google, OTP, me, preferences, delete account |
| `billing.controller.ts` | `/billing/*` | Catalog, wallet, Google verify, simulate purchase, RTDN webhook |
| `template.controller.ts` | `/templates/*` | Browse, create, publish, autofill, image generate, delete |
| `instance.controller.ts` | `/instances/*` | Create/list/archive/reset/delete, protagonist, settings, play-status, realms |
| `persona.controller.ts` | `/personas/*` | Player persona CRUD |
| `chronicle.controller.ts` | `/chronicle/*` | Events, memories, recap, calendar, bonds, kinship, relation candidates, places, side chats, edit/replay/rewind/track |
| `play-ws.controller.ts` | `/ws/play` | WebSocket upgrade and message dispatch |
| `admin.controller.ts` | `/admin/*` | Admin CRUD, continuity audits, event projections, ink grants, generation logs |

---

## Services (all 27)

Each file under `everlore-server/src/services/`. One paragraph role + key methods.

### `auth.service.ts`

Account identity and preferences. **Key methods:** `registerWithPassword`, `authenticatePassword`, `signInWithGoogle`, `sendPhoneOtp`, `signInWithPhoneOtp`, `getUserById`, `getLiveTier` (entitlement-aware), `updatePreferences`, `toJwtPayload`, `serializeUser`, plus `defaultUserPreferences()`.

### `billing.service.ts`

Ink wallet and entitlements. Balance is the sum of append-only `ink_ledger` rows; catalog defines welcome ink, tier monthly allotments, and costs (`story_turn`, autofill, image). **Key methods:** `catalog`, `ensureWelcomeInk`, `wallet`, `reserve` / `release` / `settle` (idempotent reservation lifecycle used by WS + worker), `grantAdminInk`, `simulatePurchase` (non-production), `verifyGooglePurchase`, `syncGoogleNotification`, `voidGooglePurchase`, `applyVerifiedPurchase`. See [BILLING.md](./BILLING.md).

### `google-play.service.ts`

Google Play Billing API client for product verification, subscription acknowledge, consumable consume, and RTDN bearer checks. **Key methods:** `configured`, `verifyRtdnBearer`, `verify`, `acknowledgeSubscription`, `consume`. Called by billing service; not exposed directly on routes.

### `play-ws.service.ts`

WebSocket play session: socket registry, Redis pub/sub relay to user channels, per-instance generation locks, rate limits, and action routing (chat / continue / side_chat / replay) including Ink `reserve` before `generationService` dispatch. **Key methods:** `handleOpen`, `handleMessage`, `handleClose`; helpers `setupRedisPubSub`, `disconnectUserSockets`.

### `generation.service.ts`

Thin dispatch only — validates session/consent and enqueues BullMQ jobs. Context packets are built in the **worker**, not here. **Key methods:** `dispatch` (default turn, `attempts: 2`), `dispatchSideChat` (`mode: 'side_chat'`, `attempts: 1`), `dispatchReplay` (`mode: 'replay'`, `attempts: 1`), `loadInstance` (Play bootstrap: recent events, memories, codex).

### `context-packet.service.ts`

Assembles one turn’s briefing after retrieval: recents → RAG → codex pins → time/place, plus side-chat scoped packets. **Key exports:** `buildContextPacket`, `buildSideChatPacket`.

### `instance.service.ts`

Playthrough lifecycle and Redis session cache (`session:{id}`, ~1h TTL). **Key methods:** `create`, `getById`, `getPlayStatus`, `list` / `listByTemplate` / `listRealms`, `archive`, `setPlayerProtagonist`, `listReusableProtagonists`, `loadSession`, `updateSettings`.

### `template.service.ts`

World/character template CRUD, publish, and lore embedding. **Key methods:** `create`, `update`, `publish`, `getById`, `listPublished`, `listByCreator`, `interestsFor` (discovery bias from user prefs).

### `template-cast.service.ts`

Materializes authored `seed_cast` into per-instance character cards / deltas at creation (never shared mutable cards). **Key exports:** `templateCastDeltas`, `materializeTemplateCast`.

### `persona.service.ts`

Account-level persona CRUD with limits; busts Redis sessions on instances using a deleted persona. **Key methods:** `list`, `create`, `update`, `delete` (`PERSONA_LIMITS`).

### `deletion.service.ts`

Cascade delete/reset: Mongo projections + Pinecone namespaces + Redis session bust. **Key methods:** `deleteTemplate` / `deleteTemplateById`, `resetInstance`, `deleteInstance` / `deleteInstanceById`, `deleteInstanceData`, `deleteAccount`.

### `memory.service.ts`

Chronicle reads and mutation of the event ledger: events/memories/recap/threads, memory edit/delete, event edit, streaming replay, variant select, rewind with checkpoint-aware projection rebuild, privacy gate `mainVisibleMemoryScope`. **Key methods:** `getEvents`, `getMemories`, `listThreads`, `buildRecap`, `editMemory`, `deleteMemory`, `rewindToSequence`, `editEvent`, `replayEvent`, `selectReplayVariant`.

### `character-codex.service.ts`

Codex apply/rebuild, ranking for prompt injection, protagonist seed, identity merge/rename confirmation, relationship meters/facts, character edit. **Key methods:** `listForInstance`, `findMentionedCharacters`, `seedProtagonist`, `applyDeltas`, `rebuildCodexFromLedger`, `setImmutableFacts`, `confirmIdentityRename` / `confirmIdentityMerge` / `finalizeIdentityMerge`, `applyManualIdentityRevisions`, `editCharacter`, `listRelationships`, `characterMemories`. Helpers: `rankCodexForInjection`, `applyRelationshipDeltas`, etc.

### `entity-graph.service.ts`

Entity registry, location containment spine, mention resolution, relationship/narrative edges, rewind repair, stub lifecycle. **Key methods:** `resolveOrCreateEntities`, `syncCodexEntities`, `mergeCharacterEntities`, `ensurePlayerEntity` / `ensureStubEntity`, `recordCharacterLocations`, `placeAncestry`, `characterPositions`, `ensureSceneParticipantStubs`, `archiveStaleStubs`, `wakeStubsByCues`, `syncRelationshipEdges`, `upsertNarrativeEdge`, `findEntitiesMentioned`, `resolveLocationAnchor` / `placeLocation`, `applyLocationFacts`, `listKnownLocations`, `repairAfterRewind`, and related prune/backfill helpers.

### `kinship-graph.service.ts`

Structural family graph on `entity_edges` (`type: 'kinship'`): apply assertions, co-parent derivation, lifecycle transitions, premise seed, ledger rebuild / incremental apply, relatives summaries for prompts/UI. **Key methods:** `confirmedRelationsToSelf`, `applyRelationAssertions`, `deriveCoParents`, `applyLifecycleTransitions`, `seedPremiseKinship`, `rebuildFromLedger`, `applyLedgerSince`, `relativesOf`, `kinSummary`, `kinshipBrief`. See [KINSHIP_GRAPH.md](../memory/KINSHIP_GRAPH.md).

### `relation-candidate.service.ts`

Player review queue for narrator-proposed kinship/identity revisions (not canon until accepted). **Key methods:** `propose`, `listOpen`, `getOpen`, `resolve`, `toClient`.

### `memory-supersession.service.ts`

When codex retires a fact, find and archive matching memory vectors (`status: superseded`) with lineage links. **Key method:** `supersedeMemories`.

### `location.service.ts`

Places index + per-place journal; owns `location_stats` refresh so the Places tab never full-scans. **Key methods:** `listLocations`, `refreshLocationStat`, `backfillLocationStats`, `getLocationJournal`.

### `time.service.ts`

Story calendars, timeline branches, anchors for next events, fork/switch reality, flashback re-anchor. **Key methods:** `ensureDefaultCalendar`, `rederiveDefaultCalendar`, `ensureMainTimeline`, `initialAnchor`, `anchorForNextEvent`, `timelineContext`, `listCalendar`, `forkTimeline`, `setActiveTimeline`, `updateEventTimeAnchor`.

### `side-chat.service.ts`

Private character threads: reachability check, thread list, paginated history. **Key methods:** `checkReachability`, `listThreads`, `getThread`.

### `projection-checkpoint.service.ts`

Chunked world-projection snapshots for scalable rewind/replay repair. **Key methods:** `latestBefore`, `create`, `instanceStateBefore`, `auditCheckpointDrift`, `restoreCodexAndKinship`, `pruneOld`. Driven by maintenance scheduler + rewind path in memory service.

### `continuity-audit.service.ts`

Read-only multi-check consistency report across projections (detection only). **Key method:** `audit`.

### `admin.service.ts`

Ops dashboard and paginated CRUD over users/worlds/instances/events/memories/characters/generation logs; continuity audit listing; event projection inspect. **Key methods:** `overview`, list/get/update/delete variants, `listContinuityAuditStatus`, `listGenerationLogs`, `getEventProjections`. Ink grants go through billing from the admin controller.

### `autofill.service.ts`

One-shot LLM drafts for Forge (world or character). **Key methods:** `autofillWorld`, `autofillCharacter` (billable via Ink).

### `image.service.ts`

Cover/avatar generation → temporary S3, with prompt scrubbing / decoration. **Key methods:** `generatePreview`; helpers `scrubDeviceLeakage`, `decorateImagePrompt`.

### `storage.service.ts`

S3 upload / delete / promote (temp → durable CDN URL). **Key methods:** `upload`, `delete`, `promote`, `keyFromUrl`; `isStorageConfigured()`.

### `pinecone-cleanup.service.ts`

Namespace and vector deletion helpers used by deletion/reset paths. **Key exports:** `deletePineconeNamespace`, `deletePineconeVector`.

---

## External providers (not under `services/`, but used by them)

| Module | Role |
|--------|------|
| `providers/auth.provider.ts` | Google token verify, Twilio OTP (mock in dev) |
| `providers/rag.provider.ts` | Hybrid search: vector + keyword + entity neighborhood + location + timeline + RRF; `querySummaries` |

---

## Prompt & AI utilities (hot path)

| Module | Role |
|--------|------|
| `utils/prompt-builder.ts` | Static cacheable prefix + dynamic sections + token budgets |
| `utils/prose-hygiene.ts` | Rules-first validation; optional LLM repair |
| `utils/token-counter.ts` | Cached tiktoken for budget math |
| `utils/event-window.ts` | Shared limits (prompt recents, Play load, Chronicle page) |
| `utils/generation-lock.ts` | Redis turn lock + worker heartbeat (see [WORKERS.md](./WORKERS.md)) |
| `utils/world-authority.ts` | Fact source ranking / visibility scopes |
| `ai/client.ts` | `callLLM` / `callLLMStream` |
| `ai/models.ts` | Central model IDs + env overrides |

---

## Key design choices

1. **Dispatch is thin** — RAG and codex assembly run in the worker via `context-packet.service` so retrieval precedes codex selection.
2. **Session cache** — Redis avoids re-assembling instance JSON every WS message; busted on rewind/settings/persona/deletion.
3. **Fire-and-forget on hot path** — Generation defers anomalies, signal ledger, some graph/codex compaction so streaming isn’t blocked.
4. **Events are canon** — Edits, replays, rewinds rebuild projections (codex, graph, kinship, checkpoints) from the ledger.
5. **Ink reservations** — WS reserves before enqueue; worker settles on success / after visible-stream failure, releases only on final pre-stream fail ([WORKERS.md](./WORKERS.md), [BILLING.md](./BILLING.md)).

---

## Adding a new feature

1. Model types in `src/models/` (+ `COLLECTIONS` + `mongo-indexes.ts` if needed)
2. Service method with ownership checks
3. Controller handler + route
4. If async/LLM-heavy → enqueue in `src/queues/` + processor in `worker/processors/`
5. Update [API.md](./API.md) and related docs
