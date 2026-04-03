# Everlore Frontend Architecture Overview

## What is Everlore?

Everlore is a persistent AI-driven chat RPG built with Flutter. Every player choice, character interaction, and world event is remembered in a living, evolving story world. Players interact through natural-language chat, and an AI narrator responds with narrative text, mutating world state, and curating memories.

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Framework | Flutter 3.x (Dart SDK ^3.8.1) | Cross-platform UI (iOS, Android, Web, macOS, Linux, Windows) |
| State Management | `flutter_bloc` (Cubit pattern) | Predictable, testable state with `Equatable` |
| Routing | `go_router` | Declarative, type-safe navigation with path parameters |
| Networking (REST) | `http` | Standard HTTP client for API calls |
| Networking (Real-time) | `web_socket_channel` | WebSocket communication for live gameplay |
| Auth | `google_sign_in` + `flutter_secure_storage` | Google Sign-In, backend JWT persistence, and phone OTP session storage |
| Local Storage | `sqflite` | SQLite for offline event caching |
| Secure Storage | `flutter_secure_storage` | Encrypted key-value store for auth tokens |
| Connectivity | `connectivity_plus` | Network state monitoring for reconnection logic |
| Environment | `flutter_dotenv` | `.env` file loading for configuration secrets |
| IDs | `uuid` | Unique identifier generation |

---

## Directory Structure

```
everlore/lib/
├── main.dart                    # App entry point
├── app_routes.dart              # GoRouter route definitions
├── app/
│   └── theme/
│       └── nexus_theme.dart     # Dark theme (NexusTheme)
├── core/
│   ├── auth/
│   │   ├── auth_service.dart    # Session persistence + backend auth flows
│   │   └── google_auth_service.dart # Google Sign-In wrapper
│   ├── config/
│   │   └── env.dart             # API/WS base URL config
│   ├── network/
│   │   ├── api_client.dart      # REST client with JWT auth
│   │   └── ws_manager.dart      # WebSocket singleton with reconnect
│   └── storage/
│       ├── local_db.dart        # SQLite cache for events
│       └── secure_storage.dart  # Token/user data encryption
├── features/
│   ├── home/                    # World instance list (home dashboard)
│   ├── play/                    # Real-time gameplay chat
│   ├── chronicle/               # Timeline & memory browser
│   └── templates/               # Browse & create from world templates
├── screens/
│   ├── auth_screen.dart         # Auth screen (Google Sign-In + phone OTP)
│   └── home_screen.dart         # Legacy home screen (placeholder)
├── shared/
│   ├── models/                  # Data models (User, Event, Memory, etc.)
│   └── widgets/                 # Reusable UI components
└── widgets/                     # Empty; future shared widgets
```

---

## Architectural Pattern: Feature-First + Cubit BLoC

Each feature follows the same layered structure:

```
features/<feature>/
├── data/
│   └── <feature>_repository.dart   # API/data access layer
├── presentation/
│   ├── <feature>_screen.dart       # Top-level screen widget
│   └── widgets/                    # Feature-specific UI components
└── state/
    └── <feature>_cubit.dart        # Cubit (state + business logic)
```

### Data Flow

```
┌─────────────┐    ┌──────────┐    ┌──────────────┐    ┌────────────┐
│  UI Widget   │───>│  Cubit   │───>│  Repository  │───>│ ApiClient  │
│  (Screen)    │<───│ (State)  │<───│  (Data)      │<───│ / WsManager│
└─────────────┘    └──────────┘    └──────────────┘    └────────────┘
     BlocBuilder       emit()          static methods        HTTP/WS
     BlocConsumer      StreamSub
```

1. **UI** dispatches actions to the **Cubit** via method calls
2. **Cubit** invokes **Repository** static methods
3. **Repository** uses `ApiClient` (REST) or `WsManager` (WebSocket)
4. **Cubit** emits new state, UI rebuilds via `BlocBuilder`/`BlocConsumer`

---

## Key Design Decisions

### Singleton WebSocket Manager
`WsManager` is a singleton (`factory WsManager() => _instance`) that maintains a single persistent WebSocket connection. It includes:
- Exponential backoff reconnection (up to 10 attempts)
- Offline message queuing
- Keepalive pings every 25 seconds
- Broadcast streams for event types (`onGenerationComplete`, `onMemoriesCurated`, `onError`, `onInstanceLoaded`, `onConnectionState`)
- Connectivity listener via `connectivity_plus`

### Optimistic UI Updates
When a player sends a message, an optimistic `GameEvent` is immediately appended to the event list. When the server responds with `generation_complete`, the optimistic event is replaced with the real one. This provides a responsive feel even on slow connections.

### Local SQLite Cache
`LocalDb` caches the last 50 events per instance in SQLite. When the `PlayCubit` initializes, it loads cached events immediately for a fast first paint, then replaces them with fresh server data via `instance_loaded`.

### Dark-Only Theme
The app uses a single dark theme (`NexusTheme.dark`) with a deep navy/purple palette. All colors are hardcoded (no dynamic theming yet).

---

## Navigation Map

```
/ (HomeScreen) ─── World list dashboard
├── /auth (AuthScreen) ─── Google Sign-In or phone OTP
├── /play/:instanceId (PlayScreen) ─── Real-time game chat
│   └── /chronicle/:instanceId (ChronicleScreen) ─── Timeline & memories
└── /templates (BrowseTemplatesScreen) ─── Browse world templates
    └── /templates/:templateId (TemplateDetailScreen) ─── Template details
```

All routes are defined declaratively in `app_routes.dart` using `GoRouter`.

---

## Backend API Contract

The frontend expects a backend server (default `http://localhost:3000`) with these endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/register` | Email/password registration |
| POST | `/auth/login` | Email/password login |
| POST | `/auth/google` | Google OAuth login |
| POST | `/auth/otp/send` | Send SMS verification code |
| POST | `/auth/otp/verify` | Verify SMS code and create/login session |
| GET | `/auth/me` | Get current user |
| GET | `/instances` | List user's world instances |
| POST | `/instances` | Create instance from template |
| POST | `/instances/:id/archive` | Archive an instance |
| GET | `/templates` | List published templates |
| GET | `/templates/:id` | Get template by ID |
| GET | `/chronicle/events/:instanceId` | Paginated event history |
| GET | `/chronicle/memories/:instanceId` | List memories for instance |
| PUT | `/chronicle/memory/:id` | Edit memory text/type/importance |
| DELETE | `/chronicle/memory/:id` | Delete a memory |
| PUT | `/chronicle/event/:id` | Edit event narrative/input |
| WS | `/ws/play?token=<jwt>` | Real-time gameplay WebSocket |

### Authentication Flow

1. The Flutter app loads `.env` and initializes Google Sign-In with `GOOGLE_WEB_CLIENT_ID`.
2. Google Sign-In returns a Google ID token, which the app exchanges with `POST /auth/google`.
3. Phone auth uses `POST /auth/otp/send` followed by `POST /auth/otp/verify`.
4. The backend returns the same Everlore JWT shape for Google, phone OTP, register, and login.
5. The client stores that JWT in secure storage and reuses it for both REST and WebSocket calls.

### WebSocket Message Types

**Client → Server:**
| Action | Fields | Description |
|--------|--------|-------------|
| `ping` | — | Keepalive |
| `chat` | `instance_id`, `payload.message` | Player sends message |
| `load_instance` | `instance_id` | Request instance data |

**Server → Client:**
| Type | Fields | Description |
|------|--------|-------------|
| `connected` | — | Connection established |
| `ack` | — | Message acknowledged |
| `pong` | — | Keepalive response |
| `instance_loaded` | `data.instance`, `data.template`, `data.recentEvents`, `data.memories` | Full instance state |
| `generation_complete` | `instanceId`, `event.*`, `event.state_diff` | AI narrative response |
| `memories_curated` | `instanceId`, `memories[]` | New memories created |
| `generation_failed` / `error` | `message`, `code` | Error notification |
