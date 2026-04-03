# Play Feature

The Play feature is the core gameplay experience — a real-time chat interface where players interact with the AI narrator through a WebSocket connection.

---

## File Structure

```
features/play/
├── data/                   (none — uses WsManager directly)
├── presentation/
│   ├── play_screen.dart
│   └── widgets/
│       ├── narrative_bubble.dart
│       ├── player_input.dart
│       └── world_state_bar.dart
└── state/
    └── play_cubit.dart
```

---

## PlayScreen

**File:** `lib/features/play/presentation/play_screen.dart`

Route: `/play/:instanceId`

A stateful screen that displays a chat-like interface with world state, narrative bubbles, and player input.

### State Management
- Provided with `BlocProvider<PlayCubit>` at the widget level
- Uses `BlocConsumer` for both building UI and reacting to new events (auto-scroll)

### Layout (top to bottom)

```
┌────────────────────────────────────┐
│ AppBar: Title + Connection Status  │
│         + Chronicle button         │
├────────────────────────────────────┤
│ WorldStateBar (expandable)         │  ← Shows world stats
├────────────────────────────────────┤
│ ErrorBanner (conditional)          │  ← Shows errors
├────────────────────────────────────┤
│                                    │
│ ListView of NarrativeBubble        │  ← Chat history
│                                    │
├────────────────────────────────────┤
│ PlayerInput bar                    │  ← Text field + send button
└────────────────────────────────────┘
```

### AppBar Details
- **Title:** Template title from state (or "Loading...")
- **Subtitle:** Connection indicator (green dot + "Connected" / red dot + "Disconnected")
- **Back button:** `context.pop()`
- **Chronicle button:** Navigates to `/chronicle/:instanceId` (only shown when instance is loaded)

### Auto-Scroll
When `state.events.length` changes (new event), the listener triggers `_scrollToBottom()` which animates the scroll controller to max extent after a 100ms delay.

### Loading State
When `isLoading` is true and events are empty, shows a centered `CircularProgressIndicator`. Cached events are shown immediately (from SQLite), then replaced when `instance_loaded` arrives.

---

## NarrativeBubble

**File:** `lib/features/play/presentation/widgets/narrative_bubble.dart`

Renders a single `GameEvent` as a chat bubble pair (player input + AI response).

### Layout

```
Player input (right-aligned):
┌─────────────────────────┐
│ "I attack the dragon"   │  ← Purple-tinted background, purple border
└─────────────────────────┘

AI response (left-aligned):
┌───────────────────────────────┐
│ "The dragon roars as flames   │  ← Dark navy background
│  engulf the battlefield..."   │
│ [combat]                      │  ← Scene tag with color coding
└───────────────────────────────┘

Optimistic loading:
┌───────────────────────────────┐
│ ⟳ "The world is responding..."│  ← Spinner + italic text
└───────────────────────────────┘
```

### Props

| Prop | Type | Description |
|------|------|-------------|
| `event` | `GameEvent` | The event to render |
| `onLongPress` | `VoidCallback?` | Long press handler on AI response (not currently used) |

### Conditional Rendering
- Player input bubble: shown if `event.playerInput` is non-empty
- AI response bubble: shown if `event.aiResponse` is non-empty
- Optimistic indicator: shown if `event.isOptimistic` and `event.aiResponse` is null

### Scene Tag Colors

| Tag | Color |
|-----|-------|
| `combat` | `Colors.redAccent` |
| `intimate` | `Colors.pinkAccent` |
| `exploration` | `Colors.greenAccent` |
| `existential` | `Colors.cyanAccent` |
| `cosmic` | `Colors.deepPurpleAccent` |
| `mundane` | `Colors.grey` |
| (default) | `Colors.blueAccent` |

---

## PlayerInput

**File:** `lib/features/play/presentation/widgets/player_input.dart`

A stateful text input bar with send button at the bottom of the play screen.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `isGenerating` | `bool` | Whether AI is generating (disables input) |
| `isConnected` | `bool` | WebSocket connection status |
| `onSend` | `ValueChanged<String>` | Callback with trimmed message text |

### Behavior

| State | Hint Text | Enabled | Send Button |
|-------|-----------|---------|-------------|
| Generating | "Waiting for response..." | No | Spinning indicator, grey |
| Connected | "What do you do?" | Yes | Send icon, purple |
| Disconnected | "Reconnecting..." | No | Send icon, grey (implicit) |

### Submit Triggers
- Pressing the send button (FAB)
- Pressing Enter/Return on keyboard (`TextInputAction.send`)
- Empty messages are ignored
- After sending, the text field is cleared and focus is re-requested

---

## WorldStateBar

**File:** `lib/features/play/presentation/widgets/world_state_bar.dart`

An expandable bar showing the current world state as a set of `StatBar` widgets.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `worldState` | `Map<String, num>` | Current world state values |
| `expanded` | `bool` | Whether the bar is expanded |
| `onToggle` | `VoidCallback` | Toggle expanded/collapsed |

### Behavior
- Collapsed: shows "World State" header with expand icon
- Expanded: renders each stat as a `StatBar` with formatted label (snake_case → Title Case)
- Animated expand/collapse with 300ms duration
- Background color: `Color(0xFF12122a)`

### Label Formatting
```dart
_formatLabel("health_points") → "Health Points"
_formatLabel("mana") → "Mana"
```

---

## PlayCubit

**File:** `lib/features/play/state/play_cubit.dart`

See [State Management docs](../architecture/state-management.md#playcubit) for full details.

### WebSocket Integration
The PlayCubit does not use a repository layer — it communicates directly with `WsManager` since all gameplay is real-time via WebSocket. It subscribes to 5 streams and manages the full lifecycle including cleanup on `close()`.

### State Diff Application
When `generation_complete` arrives, the instance's world state is updated via `WorldInstance.applyStateDiff()`:
```dart
instance: state.instance?.applyStateDiff(eventData['state_diff']),
```
This merges new values into the existing `worldState` and `activeFlags` maps.
