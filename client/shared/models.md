# Shared Models

Immutable Dart classes with `fromJson` factories. See `lib/shared/models/`.

**Server schemas:** [../../server/DATA_MODEL.md](../../server/DATA_MODEL.md)

---

## GameEvent (`event.dart`)

One turn in the story.

| Field | Notes |
|-------|-------|
| `sequence`, `type` | Turn order; `narration`, `travel`, `calendar_tick`, … |
| `playerInput`, `aiResponse` | Prose |
| `sceneTag` | dialogue, romantic, combat, … |
| `choices` | Tap-to-play chips — `Choice`: `label`, `kind` (`say`/`act`), `send` |
| `presentCharacters` | Canonical codex names in scene (bond rail dimming, approach vs seek out) |
| `replayVariants` | Alternative AI responses; each carries its own `choices` + `presentCharacters` |
| `milestone`, `timeAdvanced`, `fateThread` | Almanac / time-passage UI |
| `travel` | `TravelMove`: `{ from, to }` + `label` getter for headers |
| `isUserEdited` | Edited-turn indicator in bubble |
| `isTimePassage`, `isTravel` | UI header helpers |
| `isOptimistic` | Local placeholder while generating |

**Local:** `LocalDb` caches events but **does not** persist `choices` or replay variants — full data from WS/REST on load.

---

## Memory (`memory.dart`)

Curated long-term fact.

| Field | Notes |
|-------|-------|
| `text`, `type`, `importance` | 1–5 stars in UI |
| `subjects`, `objects` | Entity names for display + `concerns(name)` helper |
| `emotionalValence`, `unresolvedThread` | Rich atom UI chips |
| `isArchived` | Hidden from default Echoes |

---

## WorldInstance (`world_instance.dart`)

Active playthrough snapshot from `instance_loaded`.

| Field | Notes |
|-------|-------|
| `worldState`, `activeFlags` | RPG stats |
| `currentScene` | Tag + turn count |
| `characters` | Codex cards for bond rail |
| `meta.milestones` | Story timeline sheet |
| Session settings | POV, chat mode, reply length, focus character, persona |

---

## CharacterProfile (`character_profile.dart`)

Codex card + bond meters.

| Field | Notes |
|-------|-------|
| `relationship` | `RelationshipMeters`: trust, affection, fear, rivalry (0–100) |
| `immutableFacts`, `mutableState` | Canon strings |
| `dispositionToPlayer`, `hiddenThought` | Narrative (hidden not shown in Bonds API) |
| `isProtagonist` | Player/sentient persona |

---

## WorldTemplate (`world_template.dart`)

Creator/discovery blueprint: seed, lore, stats, flags, cover URL, `isSentient`, genres.

---

## User (`user.dart`)

Auth user + `UserPreferences` (nsfw, theme). Includes `interests`, `playerName`, `playerGender` for onboarding.

---

## Persona (`persona.dart`)

Reusable player identity overlay selected per instance.

---

## RealmPlayStatus (`realm_play_status.dart`)

Multi-playthrough picker: template → list of instances with last-played metadata.

---

## Chronicle DTOs (in `features/chronicle/data/`)

Not in `shared/models/` but used by Lore Tome:

| File | Types |
|------|-------|
| `recap_data.dart` | RecapData, RecapBond, RecapThread |
| `calendar_data.dart` | StoryCalendar, TimelineBranch, CalendarEvent |
| `location_journal.dart` | `LocationPlace`, `LocationJournal` (`permanentFacts`, `currentState`) |
| `relationship_ledger.dart` | RelationshipLedger, BondMoment |
| `threads_data.dart` | StoryThread, ThreadsData |
| `side_chat_data.dart` | SideChatThread, SideChatTurn |

All use `Equatable` + `fromJson`.
