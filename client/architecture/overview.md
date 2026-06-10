# Everlore Frontend — Architecture Overview

Flutter app for AI narrative RPG play, world creation, and the Lore Tome (Chronicle).

**Product walkthrough:** [../../system-guide/04-frontend-where-things-live.md](../../system-guide/04-frontend-where-things-live.md)  
**File-by-file reference:** [../../code-reference/CLIENT.md](../../code-reference/CLIENT.md)

---

## Technology stack

| Layer | Technology |
|-------|------------|
| Framework | Flutter 3.x (Dart ^3.8) |
| State | `flutter_bloc` Cubits + `Equatable` |
| Routing | `go_router` — shell nav + root pushes |
| REST | `http` via `ApiClient` |
| Real-time | `web_socket_channel` via `WsManager` singleton |
| Auth | Google Sign-In, email/password, phone OTP → backend JWT |
| Local cache | `sqflite` (events), `flutter_secure_storage` (token) |
| Theme | `EverloreTheme` in `app/theme/nexus_theme.dart` |

---

## App structure

```text
Pre-app:     splash → welcome → auth → onboarding
Shell tabs:  Discover | Realms | Profile  (+ floating create)
Over shell:  play, chronicle, forge, template detail, realm picker
```

### Shell navigation

`ScaffoldWithNavBar` — bottom glass nav with three branches (Explore, Realms, Profile). Play and Chronicle push **over** the shell on the root navigator.

---

## Directory layout

```text
lib/
├── main.dart, app_routes.dart
├── app/theme/everlore_theme (nexus_theme.dart)
├── core/
│   ├── auth/          Session, Google, JWT persist
│   ├── config/env.dart
│   ├── network/       ApiClient, WsManager
│   └── storage/       SecureStore, LocalDb
├── features/
│   ├── play/          Real-time game
│   ├── chronicle/     Lore Tome (7 tabs)
│   ├── home/          Realms tab
│   ├── templates/     Discover browse
│   ├── creator/       Forge world/character
│   └── personas/      Player personas
├── screens/           Splash, auth, onboarding, discover
└── shared/
    ├── models/        Event, Memory, WorldInstance, …
    └── widgets/       Nav bar, neu auth, loaders
```

---

## Data flow

```text
UI (Screen/Widget)
    → Cubit (PlayCubit, ChronicleCubit, …)
        → WsManager (play, side chat)  OR  Repository (REST)
            → everlore-server
```

Play is **WebSocket-first**. Chronicle is **REST-first** (lazy per tab). Side chat uses both.

---

## Key design decisions

1. **Bounded client state** — Play keeps last 100 events / 50 memories; older history via Chronicle pagination.
2. **Singleton WebSocket** — One connection shared; `load_instance` resyncs after rewind.
3. **Lazy Chronicle loads** — Seven tabs don't all fetch on open.
4. **Local event cache** — SQLite shows cached turns instantly while WS reconnects.
5. **No client-side prompt building** — All memory/RAG/codex logic is server-side.

---

## Related docs

| Doc | Topic |
|-----|-------|
| [routing.md](./architecture/routing.md) | All routes |
| [state-management.md](./architecture/state-management.md) | Cubits |
| [core-layer.md](./core/core-layer.md) | Network, auth, storage |
| [play-feature.md](./features/play-feature.md) | Game screen |
| [chronicle-feature.md](./features/chronicle-feature.md) | Lore Tome |
| [shared/models.md](./shared/models.md) | Data models |
