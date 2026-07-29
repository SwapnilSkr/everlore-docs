# Chronicle Feature (Lore Tome)

The **Lore Tome** is the player's library over their story: seven tabs plus drill-down screens for places, character memories, and private side chats. Play’s **World Actions** also call Chronicle kinship / relation-candidate REST APIs (documented below).

**Route:** `/chronicle/:instanceId` (`?section=` optional)  
**See also:** [../../system-guide/04-frontend-where-things-live.md](../../system-guide/04-frontend-where-things-live.md) · server [API.md](../../server/API.md) · [KINSHIP_GRAPH.md](../../memory/KINSHIP_GRAPH.md)

---

## File structure

```
features/chronicle/
├── data/
│   ├── chronicle_repository.dart   # REST API client (incl. kinship / candidates)
│   ├── calendar_data.dart
│   ├── location_journal.dart
│   ├── relationship_ledger.dart
│   ├── recap_data.dart
│   ├── side_chat_data.dart
│   └── threads_data.dart
├── presentation/
│   ├── chronicle_screen.dart       # Tab shell
│   ├── location_journal_screen.dart
│   ├── character_memory_screen.dart
│   ├── side_chat_screen.dart
│   └── widgets/
│       ├── recap_view.dart
│       ├── almanac_view.dart
│       ├── places_view.dart
│       ├── bonds_view.dart
│       ├── threads_view.dart
│       ├── echoes_filter_bar.dart
│       ├── memory_card.dart
│       └── edit_dialog.dart
└── state/
    └── chronicle_cubit.dart
```

Shared DTO used by Play World Actions: `shared/models/relation_candidate.dart`.

---

## ChronicleScreen

Opens on **Recap** tab; calls `loadRecap()` on create.

### Seven tabs (horizontal scroll)

| Tab | Widget | Loads on first visit | API |
|-----|--------|----------------------|-----|
| **Recap** | `RecapView` | Default | `GET /chronicle/recap/:id` |
| **Timeline** | `NarrativeBubble` list (travel turns show route header) | If events empty | `GET /chronicle/events/:id` |
| **Echoes** | `MemoryCard` + filters | If memories empty | `GET /chronicle/memories/:id` |
| **Almanac** | `AlmanacView` | If calendar null | `GET /chronicle/calendar/:id` |
| **Places** | `PlacesView` | If locations null | `GET /chronicle/locations/:id` |
| **Bonds** | `BondsView` | If bonds null | `GET /chronicle/relationships/:id` |
| **Threads** | `ThreadsView` | If threads null | `GET /chronicle/threads/:id` |

Tab switches call `ChronicleCubit.switchTab()` which lazy-loads missing data.

---

## Drill-down screens

| Screen | Entry | API |
|--------|-------|-----|
| `LocationJournalScreen` | Tap place in Places tab | Sections: **What is true of this place** (`permanentFacts`), **How it stands now** (`currentState`), moments + memories |
| `CharacterMemoryScreen` | Tap bond card | `GET /chronicle/relationships/:id/:charId/memories` |
| `SideChatScreen` | Private chat icon on bond card | REST history + WS `side_chat` |

Side chat is **not** in the main Timeline — separate thread surface.

---

## Kinship & relation candidates (World Actions)

Used by Play’s `WorldActionsButton` via `PlayCubit` → `ChronicleRepository`. These are **not** Lore Tome tabs; they are authorial / review APIs over the kinship graph and narrator proposal queue.

| Method | Endpoint | Role |
|--------|----------|------|
| `getConfirmedKinship` | `GET /chronicle/kinship/:instanceId` | Map of confirmed ties to self (character → relation label) for the Set Relationship UI |
| `setKinship` | `POST /chronicle/kinship/:instanceId` | Player-authored kinship write — body: `character`, `relation`, `correction`, optional `replaces_relation` |
| `getRelationCandidates` | `GET /chronicle/relation-candidates/:instanceId` | Open review items (`RelationCandidate`: `kind`, names, `relation`, `evidence`, …) |
| `resolveRelationCandidate` | `POST /chronicle/relation-candidate/:candidateId/resolve` | Body: `action` = `accept` \| `reject` \| `defer`, optional `relation` on accept |

**Candidate kinds** (server): `kinship`, `identity_rename`, `identity_merge`, `kinship_revision`. Candidates are **not canon** until the player accepts.

**Related:** `trackEntity` → `POST /chronicle/track/:instanceId` promotes a prose-visible character into the codex and may assert kinship (bond sheet / track flows).

Server detail: [API.md — Kinship & relation review](../../server/API.md), [DATA_MODEL.md](../../server/DATA_MODEL.md) (`entity_edges` type `kinship`, `relation_candidates`).

---

## Echoes tab

- **Search** — submit triggers `q` param (full-text on server)
- **Filter chips** — Unresolved, Important (≥4), memory types
- **Edit/delete** — `EditMemoryDialog` → `PUT/DELETE /chronicle/memory/:id`

---

## ChronicleRepository

All Chronicle REST calls. Key methods:

| Method | Endpoint |
|--------|----------|
| `getEvents` | `/chronicle/events/:instanceId` |
| `getMemories` | `/chronicle/memories/:instanceId` |
| `getRecap` | `/chronicle/recap/:instanceId` |
| `getThreads` | `/chronicle/threads/:instanceId` |
| `getRelationships` | `/chronicle/relationships/:instanceId` |
| `getCharacterMemories` | `/chronicle/relationships/:id/:charId/memories` |
| `getConfirmedKinship` / `setKinship` | `/chronicle/kinship/...` |
| `getRelationCandidates` / `resolveRelationCandidate` | `/chronicle/relation-candidates/...`, `/chronicle/relation-candidate/:id/resolve` |
| `getLocations` / `getLocationJournal` | `/chronicle/locations/...` |
| `getCalendar` / `setActiveTimeline` | `/chronicle/calendar/...` |
| `getSideChatThread` | `/chronicle/side-chats/...` |
| `trackEntity` | `POST /chronicle/track/:instanceId` |
| `rewind` | `POST /chronicle/rewind/:instanceId` |
| `editEvent` / `editMemory` / `editCharacter` | PUT routes |

### `editEvent` return type

When `ai_response` changes, the PUT response includes regenerated chips and presence. `ChronicleRepository.editEvent` returns:

```dart
({List<Choice> choices, List<String> presentCharacters})?  // null if only player_input edited
```

PlayCubit applies these onto the settled event without a refetch.

---

## ChronicleCubit

- **Lazy loading** per tab — avoids fetching all seven surfaces at once
- **Optimistic** memory edit/delete in Echoes
- **Filter state** for Echoes (`setMemoryFilters` → reload with query params)

---

## Privacy note

Timeline, Echoes, Recap, Threads, Places, and Calendar show **main-story** data only. Side-chat content appears only in `SideChatScreen`. Server enforces this; client relies on correct endpoints.
