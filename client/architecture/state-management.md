# State Management

Everlore uses the **Cubit** pattern from `flutter_bloc` for state management. Each feature has its own Cubit with an accompanying immutable state class that extends `Equatable`.

---

## Pattern Overview

```
Cubit<StateClass>
  ├── StateClass (immutable, extends Equatable)
  │     ├── data fields
  │     └── copyWith() method
  └── business methods that call emit(state.copyWith(...))
```

### Why Cubit over full BLoC?
- Simpler API: methods instead of events
- Less boilerplate for straightforward state mutations
- Same `BlocBuilder` / `BlocConsumer` integration
- Full testability via `Equatable`-based state comparison

---

## Cubits in the App

### HomeCubit

**File:** `lib/features/home/state/home_cubit.dart`

**State:** `HomeState`
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `instances` | `List<WorldInstance>` | `[]` | User's active world instances |
| `isLoading` | `bool` | `false` | Loading indicator |
| `error` | `String?` | `null` | Error message |

**Methods:**
| Method | Triggers | Description |
|--------|----------|-------------|
| `loadInstances()` | API `GET /instances` | Fetches all instances, populates list |
| `createInstance(templateId)` | API `POST /instances` | Creates new instance, prepends to list |
| `archiveInstance(instanceId)` | API `POST /instances/:id/archive` | Archives and removes from list |

**Usage:**
```dart
BlocProvider(
  create: (_) => HomeCubit()..loadInstances(),
  child: HomeView(),
)
```

---

### PlayCubit

**File:** `lib/features/play/state/play_cubit.dart`

**State:** `PlayState`
| Field | Type | Description |
|-------|------|-------------|
| `instance`, `template` | World models | Current playthrough |
| `events`, `memories`, `characters` | Lists | Active feed data |
| `isGenerating`, `isConnected`, `isLoading` | bool | UI flags |
| `error` | `String?` | Error banner |
| `totalEvents`, `hasOlderEvents` | int/bool | Pagination hints from `eventWindow` |
| `replayingEventId` | `String?` | Turn currently streaming a replay |
| `lastStatDeltas` | `Map?` | Animated stat bar deltas |
| `lastMilestone`, `milestoneStamp`, `milestones` | | Milestone toast + timeline sheet |

**Initialization Flow:**
1. Constructor calls `_init()`
2. `_loadCachedEvents()` — SQLite cache (choices/replay variants not persisted locally)
3. `_ws.connect(token, force: true)` + `loadInstance`
4. Subscribes to **12** WebSocket streams: `onInstanceLoaded`, `onGenerationDelta`, `onGenerationStreamEnd`, `onGenerationComplete`, `onMemoriesCurated`, `onCharacterCodexUpdated`, `onMilestoneUnlocked`, `onReplayDelta`, `onReplayComplete`, `onError`, `onConnectionState`

**Key methods:**
| Method | Description |
|--------|-------------|
| `sendMessage` | Optimistic turn + `chat` WS |
| `continueStory({advance})` | Quiet continue or calendar tick |
| `replayLastResponse` | WS replay of latest turn |
| `selectReplayVariant` | Local preview of variant chips/presence; commits via REST before next send |
| `editAiResponse` | REST edit; swaps server-returned chips when narrative changed |

**Variant commit:** Browsing replay arrows is local-only. `POST /chronicle/replay/select` runs in `_flushPendingVariant()` before the next player message.

**Cleanup:** All stream subscriptions cancelled in `close()`.

---

### ChronicleCubit

**File:** `lib/features/chronicle/state/chronicle_cubit.dart`

**Tabs (`ChronicleTab`):** `recap`, `timeline`, `echoes`, `almanac`, `places`, `bonds`, `threads` — default **recap**.

**State highlights:** Per-tab lazy-loaded data (`recap`, `events`, `memories`, `calendar`, `locations`, `bonds`, `threads`), Echoes filter params, loading/error.

**Methods:** `loadRecap`, `loadEvents`, `loadMemories`, `loadCalendar`, `loadLocations`, `loadBonds`, `loadThreads`, `setMemoryFilters`, `editMemory`, `deleteMemory`, `editEvent`, `switchTab` (lazy-loads missing tab data).

See [chronicle-feature.md](../features/chronicle-feature.md).

---

## Equatable Usage

All state classes extend `Equatable` and override `props` to include all mutable fields. This ensures:
- `BlocBuilder` only rebuilds when state actually changes
- `BlocConsumer.listenWhen` works correctly
- State comparisons in tests are value-based, not reference-based

```dart
@override
List<Object?> get props => [instances, isLoading, error];
```

---

## Adding a New Cubit

1. Create `lib/features/<name>/state/<name>_cubit.dart`
2. Define a state class extending `Equatable` with `copyWith()`
3. Define a Cubit class extending `Cubit<StateClass>`
4. Provide it via `BlocProvider` in the screen widget
5. Consume it via `BlocBuilder` or `BlocConsumer` in child widgets
