# Home Feature

The Home feature is the main dashboard where players view their active world instances.

---

## File Structure

```
features/home/
├── data/
│   └── home_repository.dart
├── presentation/
│   ├── home_screen.dart
│   └── widgets/
│       └── world_card.dart
└── state/
    └── home_cubit.dart
```

---

## HomeScreen

**File:** `lib/features/home/presentation/home_screen.dart`

The root screen at route `/`. Displays a list of the player's world instances as cards.

### State Management
- Provided with `BlocProvider<HomeCubit>` at the widget level
- Automatically calls `loadInstances()` on creation

### UI States

| State | UI |
|-------|-----|
| Loading + empty instances | Purple `CircularProgressIndicator` centered |
| Empty instances | Empty state with book icon, "No worlds yet" message, "Browse Worlds" button |
| Loaded | `RefreshIndicator` wrapping `ListView` of `WorldCard` widgets |
| Error | Handled by cubit (error stored in state, not currently displayed in UI) |

### AppBar Actions
| Icon | Action | Route |
|------|--------|-------|
| `Icons.explore` | Browse world templates | `/templates` |
| `Icons.person_outline` | Auth/Profile | `/auth` |

### Pull to Refresh
Wraps the list in `RefreshIndicator` calling `HomeCubit.loadInstances()`.

---

## WorldCard

**File:** `lib/features/home/presentation/widgets/world_card.dart`

A card displaying summary information about a single `WorldInstance`.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `instance` | `WorldInstance` | The world instance data |
| `onTap` | `VoidCallback` | Navigate to play screen |
| `onArchive` | `VoidCallback?` | Show archive confirmation dialog |

### Display Elements

| Element | Source | Description |
|---------|--------|-------------|
| Icon | `instance.template['is_sentient']` | Psychology icon (sentient) or AutoStories icon (game master) |
| Title | `instance.template['title']` | World template title |
| Description | `instance.template['description']` | Up to 2 lines, ellipsed |
| Events chip | `instance.meta.totalEvents` | "{N} events" badge |
| Memories chip | `instance.meta.totalMemories` | "{N} memories" badge |
| Time ago | `instance.meta.lastActiveAt` | Relative time (e.g., "2d ago", "just now") |
| Archive button | `onArchive` callback | Shows confirmation dialog before archiving |

### Archive Confirmation
When the archive button is pressed, an `AlertDialog` appears with "Cancel" and "Archive" options. On confirm, `HomeCubit.archiveInstance()` is called.

---

## HomeRepository

**File:** `lib/features/home/data/home_repository.dart`

Static data access layer for world instances.

### Methods

| Method | API Call | Returns | Description |
|--------|----------|---------|-------------|
| `getInstances({includeArchived})` | `GET /instances?include_archived=false` | `List<WorldInstance>` | Fetches user's instances with optional archived |
| `createInstance(templateId)` | `POST /instances` with `{template_id}` | `WorldInstance` | Creates new instance from template |
| `archiveInstance(instanceId)` | `POST /instances/:id/archive` | `void` | Marks instance as archived |

### Request/Response Format

**Create Instance Request:**
```json
{ "template_id": "abc123" }
```

**Create Instance Response:**
```json
{ "instance": { "_id": "...", "template_id": "...", "world_state": {}, ... } }
```

---

## HomeCubit

**File:** `lib/features/home/state/home_cubit.dart`

See [State Management docs](../architecture/state-management.md#homecubit) for full details.

### Key Behaviors
- `createInstance()` prepends the new instance to the list (most recent first)
- `archiveInstance()` filters the instance out of the local list immediately (optimistic)
- All methods set `error` on failure but do not show error UI (future improvement)
