# Chronicle Feature

The Chronicle feature provides a historical view of game events and curated memories for a specific world instance. It has two tabs: **Timeline** (event history) and **Memories** (AI-curated memory browser).

---

## File Structure

```
features/chronicle/
├── data/
│   └── chronicle_repository.dart
├── presentation/
│   ├── chronicle_screen.dart
│   └── widgets/
│       ├── edit_dialog.dart
│       └── memory_card.dart
└── state/
    └── chronicle_cubit.dart
```

---

## ChronicleScreen

**File:** `lib/features/chronicle/presentation/chronicle_screen.dart`

Route: `/chronicle/:instanceId`

### State Management
- Provided with `BlocProvider<ChronicleCubit>` at the widget level
- Calls `loadEvents()` on creation
- Uses `BlocBuilder` for UI rebuilds

### Layout

```
┌────────────────────────────────────┐
│ AppBar: "Chronicle" title          │
├────────────────────────────────────┤
│ [Timeline]  [Memories]  ← Tab bar  │
├────────────────────────────────────┤
│                                    │
│  Tab content (ListView)            │
│                                    │
└────────────────────────────────────┘
```

### Tab Navigation
Custom tab buttons in the AppBar's `bottom` slot. Active tab has a purple bottom border. Tapping switches tab and lazy-loads data.

| Tab | Content | Data Source |
|-----|---------|-------------|
| Timeline | `NarrativeBubble` list (reused from Play) | `ChronicleRepository.getEvents()` |
| Memories | `MemoryCard` list | `ChronicleRepository.getMemories()` |

### Empty States
- Timeline empty: "No events yet" centered text
- Memories empty: "No memories yet" centered text

---

## MemoryCard

**File:** `lib/features/chronicle/presentation/widgets/memory_card.dart`

Displays a single `Memory` with its type, importance rating, text content, and action buttons.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `memory` | `Memory` | Memory data |
| `onEdit` | `VoidCallback?` | Opens edit dialog |
| `onDelete` | `VoidCallback?` | Shows delete confirmation |

### Layout

```
┌────────────────────────────────────────┐
│ [relationship] ★★★★★    [✏️] [🗑️]     │  ← Type badge + importance + actions
│                                        │
│ "The blacksmith swore an oath to..."   │  ← Memory text
│                                        │
│ (Archived)                             │  ← If archived
└────────────────────────────────────────┘
```

### Memory Type Colors

| Type | Color |
|------|-------|
| `relationship` | `Colors.pinkAccent` |
| `promise` | `Colors.amberAccent` |
| `lore` | `Colors.cyanAccent` |
| `observation` | `Colors.blueAccent` |
| `emotion` | `Colors.purpleAccent` |
| `secret` | `Colors.redAccent` |
| (default) | `Colors.grey` |

### Importance Display
Renders `memory.importance` number of star icons (★).

### Delete Confirmation
Shows an `AlertDialog` with "Cancel" and "Delete" options. The dialog warns "This memory will be permanently forgotten."

---

## EditMemoryDialog

**File:** `lib/features/chronicle/presentation/widgets/edit_dialog.dart`

A dialog for editing a memory's text, type, and importance.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `initialText` | `String` | Current memory text |
| `initialType` | `String` | Current memory type |
| `initialImportance` | `int` | Current importance (1-5) |

### Form Fields

| Field | Widget | Options/Range |
|-------|--------|---------------|
| Text | `TextField` (4 lines max) | Free text |
| Type | `DropdownButtonFormField<String>` | `relationship`, `promise`, `lore`, `observation`, `emotion`, `secret` |
| Importance | `Slider` | 1-5, integer steps |

### Return Value
On "Save", pops with `Map<String, dynamic>`:
```dart
{ 'text': 'Updated text', 'type': 'lore', 'importance': 4 }
```
On "Cancel", pops with `null`.

---

## ChronicleRepository

**File:** `lib/features/chronicle/data/chronicle_repository.dart`

### Methods

| Method | API Call | Returns | Description |
|--------|----------|---------|-------------|
| `getEvents(instanceId, {page, limit, type})` | `GET /chronicle/events/:instanceId?page=N&limit=N` | `Map` with `events`, `total`, `page` | Paginated event list |
| `getMemories(instanceId, {includeArchived})` | `GET /chronicle/memories/:instanceId?include_archived=false` | `List<Memory>` | Memory list |
| `editMemory(memoryId, {text, type, importance})` | `PUT /chronicle/memory/:id` | `void` | Updates memory fields |
| `deleteMemory(memoryId)` | `DELETE /chronicle/memory/:id` | `void` | Permanently deletes memory |
| `editEvent(eventId, {aiResponse, playerInput})` | `PUT /chronicle/event/:id` | `void` | Edits event narrative or input |

---

## ChronicleCubit

**File:** `lib/features/chronicle/state/chronicle_cubit.dart`

See [State Management docs](../architecture/state-management.md#chroniclecubit) for full details.

### Key Behaviors
- **Tab switching** is lazy — only loads data when the target list is empty
- **Memory editing** optimistically updates the local list via `map()` instead of re-fetching
- **Memory deletion** optimistically removes from local list
- **Event editing** re-fetches the current page to get server-confirmed data
