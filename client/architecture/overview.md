# Everlore Frontend — Architecture Overview

Flutter app for AI narrative RPG play, world creation, Membership & Ink, and the Lore Tome (Chronicle).

**Product walkthrough:** [../../system-guide/04-frontend-where-things-live.md](../../system-guide/04-frontend-where-things-live.md)  
**File-by-file reference:** [../../code-reference/CLIENT.md](../../code-reference/CLIENT.md)

---

## Technology stack

| Layer | Technology |
|-------|------------|
| Framework | Flutter 3.x (Dart ^3.8) |
| State | `flutter_bloc` Cubits + `Equatable` |
| Routing | `go_router` — 4-tab shell + root pushes |
| REST | `http` via `ApiClient` |
| Real-time | `web_socket_channel` via `WsManager` singleton |
| Auth | Google Sign-In, email/password, phone OTP → backend JWT |
| Billing | `in_app_purchase` → server `/billing/*` (Google Play verify) |
| Local cache | `sqflite` (events), `flutter_secure_storage` (token) |
| Theme | `EverloreTheme` in `app/theme/nexus_theme.dart` |

---

## App structure

```text
Pre-app:     splash → welcome → auth → onboarding
Shell tabs:  Explore (/discover) | Realms (/) | Worlds (/my-worlds) | Personas (/personas)
             + center Create → forge world | create character
Over shell:  play, chronicle, realm overview, forge, template detail,
             realm picker, membership, profile
```

### Shell navigation

`ScaffoldWithNavBar` — floating glass nav with **four** branches (Explore, Realms, Worlds, Personas). Play, Chronicle, Realm overview, Forge, and Membership push **over** the shell on the root navigator.

---

## Directory layout

```text
lib/
├── main.dart, app_routes.dart
├── app/theme/          EverloreTheme (nexus_theme.dart)
├── core/
│   ├── auth/           Session, Google, JWT persist
│   ├── config/env.dart
│   ├── network/        ApiClient, WsManager
│   └── storage/        SecureStore, LocalDb
├── features/
│   ├── play/           Real-time game + World Actions + RealmScreen
│   ├── chronicle/      Lore Tome (7 tabs) + kinship / relation-candidates REST
│   ├── billing/        Membership & Ink
│   ├── home/           Realms tab
│   ├── templates/      Discover browse
│   ├── creator/        Forge world/character
│   └── personas/       Player personas
├── screens/            Splash, auth, onboarding, discover
└── shared/
    ├── models/         Event, Memory, WorldInstance, RelationCandidate, …
    └── widgets/        Nav bar, neu auth, loaders
```

---

## Data flow

```text
UI (Screen/Widget)
    → Cubit (PlayCubit, ChronicleCubit, …)  OR  BillingRepository
        → WsManager (play, side chat, world_action)  OR  Repository (REST)
            → everlore-server
```

Play is **WebSocket-first** (`chat`, `continue`, `world_action`, `replay`, `side_chat`). Chronicle is **REST-first** (lazy per tab). Kinship writes and relation-candidate review also go through Chronicle REST (used by World Actions). Billing is REST + Play Billing library.

---

## Key design decisions

1. **Bounded client state** — Play keeps last 100 events / 50 memories; older history via Chronicle pagination.
2. **Singleton WebSocket** — One connection shared; `load_instance` resyncs after rewind.
3. **Structured World Actions** — Travel / time / kinship / candidate review are explicit UI commands, not “hope the model parses prose.”
4. **Lazy Chronicle loads** — Seven tabs don't all fetch on open.
5. **Local event cache** — SQLite shows cached turns instantly while WS reconnects.
6. **Server-authoritative Ink** — Wallet from `GET /billing/me`; client never trusts local balances.
7. **No client-side prompt building** — All memory/RAG/codex logic is server-side.

---

## Related docs

| Doc | Topic |
|-----|-------|
| [routing.md](./routing.md) | All routes |
| [state-management.md](./state-management.md) | Cubits |
| [../core/core-layer.md](../core/core-layer.md) | Network, auth, storage |
| [../features/play-feature.md](../features/play-feature.md) | Game screen |
| [../features/chronicle-feature.md](../features/chronicle-feature.md) | Lore Tome |
| [../features/billing-feature.md](../features/billing-feature.md) | Membership & Ink |
| [../shared/models.md](../shared/models.md) | Data models |
