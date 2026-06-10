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
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `instance` | `WorldInstance?` | `null` | Current world instance with state |
| `template` | `WorldTemplate?` | `null` | Template metadata for display |
| `events` | `List<GameEvent>` | `[]` | Chat event history (narrative + player) |
| `memories` | `List<Memory>` | `[]` | Curated memories for this instance |
| `isGenerating` | `bool` | `false` | Whether AI is currently generating a response |
| `isConnected` | `bool` | `false` | WebSocket connection status |
| `isLoading` | `bool` | `true` | Initial data loading state |
| `error` | `String?` | `null` | Error message |

**Initialization Flow:**
1. Constructor calls `_init()`
2. `_loadCachedEvents()` — loads up to 50 events from local SQLite cache
3. `_ws.loadInstance(instanceId)` — sends WebSocket `load_instance` action
4. Subscribes to 5 WebSocket streams:
   - `onInstanceLoaded` — full state sync (instance, template, events, memories)
   - `onGenerationComplete` — AI narrative response + state diff
   - `onMemoriesCurated` — new memories added by AI
   - `onError` — generation failures, general errors
   - `onConnectionState` — connection status toggling

**Methods:**
| Method | Description |
|--------|-------------|
| `sendMessage(message)` | Creates optimistic event, sends via WebSocket |
| `clearError()` | Clears error banner |

**Optimistic Update Flow:**
```
User types "I attack the dragon"
  → GameEvent.optimistic() created (id: optimistic_*, sequence: -1)
  → emit([...events, optimisticEvent], isGenerating: true)
  → _ws.sendChatMessage(instanceId, message)
  
Server responds with generation_complete
  → New GameEvent created from server data
  → LocalDb.insertEvent(newEvent) — persisted to SQLite
  → emit([...events, newEvent], isGenerating: false, instance: updated)
```

**Cleanup:**
All 5 `StreamSubscription`s are cancelled in `close()`.

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
