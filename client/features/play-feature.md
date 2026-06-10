# Play Feature

Real-time gameplay — chat with the AI narrator over WebSocket, with bond rail, choices, time skips, and in-prose memory links.

**Route:** `/play/:instanceId`  
**See also:** [../../system-guide/04-frontend-where-things-live.md](../../system-guide/04-frontend-where-things-live.md)

---

## File structure

```
features/play/
├── presentation/
│   ├── play_screen.dart
│   └── widgets/
│       ├── narrative_bubble.dart
│       ├── player_input.dart
│       ├── world_state_bar.dart
│       ├── bond_rail.dart
│       ├── bond_meters.dart
│       ├── choice_chips.dart
│       ├── advance_time_button.dart
│       ├── milestone_toast.dart
│       └── story_timeline_sheet.dart
└── state/
    └── play_cubit.dart
```

No `data/` layer — Play uses `WsManager` directly for real-time gameplay.

---

## PlayScreen layout

```text
┌ Back          connection ●                    ⋮ Menu ─┐
│ WorldStateBar (if stats exist)                        │
│ BondRail (if bonded NPCs)                             │
│ Error banner (if any)                                 │
│ ── NarrativeBubble list ──                            │
│ ChoiceChips (latest turn, optional)                   │
│ PlayerInput + AdvanceTimeButton (when composer empty)   │
└───────────────────────────────────────────────────────┘
```

### Realm menu (⋮)

| Item | Action |
|------|--------|
| Chronicle | Navigate to Lore Tome |
| Story Timeline | Milestones sheet |
| Thoughts | Cast, focus character, edit cards |
| Settings | POV, mode, length, persona, reset, delete |

---

## Key widgets

### NarrativeBubble

- Player bubble (right) + AI bubble (left)
- **Gold underline** on character names → bond action sheet
- **Dotted underline** on places/things → in-play memory lens
- Travel / time-passage headers when event type matches
- Continue + Replay on latest settled turn
- Replay variant arrows when multiple variants exist

### BondRail

Up to 5 side characters with relationship ring (trust/affection/fear/rivalry). Dimmed when not in current scene.

### ChoiceChips

Server-suggested say/action chips; tap prefills composer (player edits before send).

### AdvanceTimeButton

Bottom sheet: quiet moment, hours, day, season → `continueStory(advance: ...)`.

### WorldStateBar

Expandable RPG stat HUD with animated deltas.

---

## PlayCubit

Central orchestrator. Subscribes to **12+ WS streams**.

| Outbound WS | When |
|-------------|------|
| `load_instance` | Connect / after rewind |
| `chat` | Player sends message |
| `continue` | Continue or time skip |
| `replay` | Regenerate last AI reply |

| Inbound WS | Effect |
|------------|--------|
| `instance_loaded` | Seed events, memories (50 cap), characters, milestones |
| `generation_delta/complete` | Stream + finalize turn |
| `memories_curated` | Append new memories live |
| `character_codex_updated` | Refresh bond meters |
| `milestone_unlocked` | Toast + timeline |

### Local optimizations

- **Trim caps:** 100 events, 50 memories in active state
- **SQLite cache** via `LocalDb` for fast resume (200 events)
- **Stream reveal** timers + generation/replay watchdogs
- **Optimistic** turn while waiting for first delta

### REST (via ChronicleRepository / HomeRepository)

| Action | Endpoint |
|--------|----------|
| Edit AI response | `PUT /chronicle/event/:id` |
| Rewind | `POST /chronicle/rewind/:id` + WS reload |
| Select replay variant | `POST /chronicle/replay/select/:id` |
| Edit character | `PUT /chronicle/character/:id` |
| Set protagonist | `POST /instances/:id/protagonist` |
| Update settings | `PATCH /instances/:id/settings` |

---

## In-play memory lens

Tap a name/place in prose → filters local `memories` list (no API). For full history use Lore Tome → Bonds or Echoes.

---

## Protagonist onboarding

GM worlds without a player character show onboarding to set name via `POST /instances/:id/protagonist`.
