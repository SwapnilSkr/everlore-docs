# Play Feature

Real-time gameplay — chat with the AI narrator over WebSocket, with bond rail, choices, **World Actions** (time, travel, kinship, relation-candidate review), and in-prose memory links.

**Route:** `/play/:instanceId`  
**Realm overview:** `/realm/:instanceId` (`RealmScreen` — presentation-only map of the open playthrough)  
**See also:** [../../system-guide/04-frontend-where-things-live.md](../../system-guide/04-frontend-where-things-live.md)

---

## File structure

```
features/play/
├── presentation/
│   ├── play_screen.dart
│   ├── realm_screen.dart              # Calm realm map (args from Play)
│   └── widgets/
│       ├── narrative_bubble.dart
│       ├── player_input.dart          # Composer + WorldActionsButton when empty
│       ├── world_actions_button.dart  # Continue / time / travel / kinship / candidates
│       ├── bond_rail.dart
│       ├── bond_meters.dart
│       ├── choice_chips.dart
│       ├── world_state_bar.dart
│       ├── advance_time_button.dart   # Legacy widget; play UI uses WorldActionsButton
│       ├── milestone_toast.dart
│       └── story_timeline_sheet.dart
└── state/
    └── play_cubit.dart
```

No `data/` layer — Play uses `WsManager` for real-time gameplay and `ChronicleRepository` for kinship / candidates / locations / edits.

---

## PlayScreen layout

```text
┌ Back          connection ●          realm / ⋮ Menu ─┐
│ WorldStateBar (if stats exist)                        │
│ BondRail (if bonded NPCs)                             │
│ Error banner (if any)                                 │
│ ── NarrativeBubble list ──                            │
│ ChoiceChips (latest turn, optional)                   │
│ PlayerInput + WorldActionsButton (when composer empty)│
└───────────────────────────────────────────────────────┘
```

### Realm menu (⋮)

| Item | Action |
|------|--------|
| Chronicle | Navigate to Lore Tome |
| Story Timeline | Milestones sheet |
| Thoughts | Cast, focus character, edit cards |
| Settings | POV, mode, length, persona, reset, delete |

Header can also open **RealmScreen** (`/realm/:instanceId`) with `RealmScreenArgs` (instance, template, characters, reset/delete callbacks).

---

## World Actions

**File:** `widgets/world_actions_button.dart`  
Shown beside the composer when it is **empty** (yields to the send orb once the player types).

| Action | Mechanism |
|--------|-----------|
| **Continue story** | WS `continue` (no advance) |
| **Let time pass** | WS `continue` + `{ advance: hours\|day\|days\|season }` |
| **Travel to…** | WS `world_action` `{ kind: travel, destination, companions, time_advance? }` |
| **Set a relationship** | REST `POST /chronicle/kinship/:instanceId` via `PlayCubit.setRelationship` |
| **Review story details** | REST list + resolve relation candidates (when any open) |

Travel destinations are loaded from `GET /chronicle/locations/:id` (known places minus current). Relation candidates and confirmed kinship load from Chronicle kinship APIs (see [chronicle-feature.md](./chronicle-feature.md)).

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

Up to 5 side characters with relationship ring (trust/affection/fear/rivalry). **Dimmed** when not in current scene — presence matches **canonical name + aliases** against `GameEvent.presentCharacters` (null presence = no dimming). Tap → bond action sheet with **Here now / Elsewhere** tag; **Approach** vs **Seek out** based on scene presence.

### ChoiceChips

Server-suggested say/action chips for the **active prose variant** (primary turn, replayed variant, or post-edit). Tap prefills composer. Chips regenerate on replay and AI-response edit; variant arrows swap stored chips locally; commit on next send via `_flushPendingVariant()`.

### WorldStateBar

Expandable RPG stat HUD with animated deltas.

### RealmScreen

Presentation-only overview of cover art, scene tag, cast, and playthrough actions. Does **not** own game state — open from Play with `RealmScreenArgs`.

---

## PlayCubit

Central orchestrator. Subscribes to WsManager streams (generation, replay, memories, codex, milestones, side chat, errors, connection, …).

| Outbound WS | When |
|-------------|------|
| `load_instance` | Connect / after rewind |
| `chat` | Player sends message |
| `continue` | Continue or time skip |
| `world_action` | Structured travel (and server-supported relationship kinds) |
| `replay` | Regenerate last AI reply |
| `side_chat` | SideChatScreen private thread |

| Inbound WS | Effect |
|------------|--------|
| `instance_loaded` | Seed events, memories (50 cap), characters, milestones |
| `generation_delta` / `generation_stream_end` / `generation_complete` | Stream + finalize turn (**prose stream**, not a structured JSON response body) |
| `replay_delta` / `replay_complete` | Replay stream; chips + presence per variant |
| `memories_curated` | Append new memories live |
| `character_codex_updated` | Refresh bond meters |
| `milestone_unlocked` | Toast + timeline |
| `validation` / `error` / `generation_failed` | Error banner / billing / lock failures |

### Local optimizations

- **Trim caps:** 100 events, 50 memories in active state
- **SQLite cache** via `LocalDb` for fast resume (200 events)
- **Stream reveal** timers + generation/replay watchdogs
- **Optimistic** turn while waiting for first delta

### REST (via ChronicleRepository)

| Action | Endpoint | Notes |
|--------|----------|-------|
| Edit AI response | `PUT /chronicle/event/:id` | Returns regenerated `choices` + `present_characters` when prose changed |
| Rewind | `POST /chronicle/rewind/:id` + WS reload | |
| Select replay variant | `POST /chronicle/replay/select/:id` | Commits browsed variant before next send |
| Edit character | `PUT /chronicle/character/:id` |
| Set protagonist | `POST /instances/:id/protagonist` |
| Update settings | `PATCH /instances/:id/settings` |
| Confirmed kinship | `GET/POST /chronicle/kinship/:id` | World Actions “Set a relationship” |
| Relation candidates | `GET …/relation-candidates/:id` + `POST …/relation-candidate/:id/resolve` | Review queue |
| Known places (travel) | `GET /chronicle/locations/:id` | Destination picker |
| Track entity | `POST /chronicle/track/:id` | Promote / assert from prose |

---

## In-play memory lens

Tap a name/place in prose → filters local `memories` list (no API). For full history use Lore Tome → Bonds or Echoes.

---

## Protagonist onboarding

GM worlds without a player character show onboarding to set name via `POST /instances/:id/protagonist` (or reuse from another save of the same world).
