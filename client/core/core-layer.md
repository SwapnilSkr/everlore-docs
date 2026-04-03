# Core Layer

The `core/` directory contains infrastructure code shared across all features: authentication, configuration, networking, and storage.

---

## Table of Contents

- [Config (env.dart)](#config-envdart)
- [Auth (auth_service.dart)](#auth-auth_servicedart)
- [Network - API Client (api_client.dart)](#network---api-client-api_clientdart)
- [Network - WebSocket Manager (ws_manager.dart)](#network---websocket-manager-ws_managerdart)
- [Storage - Secure Storage (secure_storage.dart)](#storage---secure-storage-secure_storagedart)
- [Storage - Local DB (local_db.dart)](#storage---local-db-local_dbdart)

---

## Config (`env.dart`)

**File:** `lib/core/config/env.dart`

Reads configuration from Dart compile-time environment variables with sensible defaults for local development.

```dart
class AppConfig {
  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:3000',
  );
  static const String wsBaseUrl = String.fromEnvironment(
    'WS_BASE_URL',
    defaultValue: 'ws://localhost:3000',
  );
}
```

### Overriding at Build Time
```bash
flutter run --dart-define=API_BASE_URL=https://api.everlore.io --dart-define=WS_BASE_URL=wss://api.everlore.io
```

---

## Auth (`auth_service.dart`)

**File:** `lib/core/auth/auth_service.dart`

Static service class handling all authentication flows. Uses `ApiClient` for network calls and `SecureStore` for token persistence.

### Methods

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `register()` | `email`, `username`, `password` | `Future<User>` | Register new account, stores token + user data |
| `login()` | `email`, `password` | `Future<User>` | Login with credentials, stores token + user data |
| `loginWithGoogle()` | `idToken` (String) | `Future<User>` | Google OAuth via server-side verification |
| `getCurrentUser()` | — | `Future<User?>` | Validates token with `/auth/me`, returns null on failure |
| `getCachedUser()` | — | `Future<User?>` | Returns locally cached user from secure storage |
| `logout()` | — | `Future<void>` | Clears all secure storage data |
| `isLoggedIn()` | — | `Future<bool>` | Checks if a token exists in secure storage |

### Token Lifecycle
1. On login/register: token → `SecureStore.saveToken()`, user JSON → `SecureStore.saveUserData()`
2. On API calls: `ApiClient._headers()` reads token via `SecureStore.getToken()` and adds `Authorization: Bearer <token>` header
3. On logout: `SecureStore.clearAll()` removes everything

### Error Handling
- `getCurrentUser()` returns `null` on any exception (token invalid, network error)
- `getCachedUser()` returns `null` if no data or JSON parse failure
- All other methods throw `ApiException` on failure

---

## Network - API Client (`api_client.dart`)

**File:** `lib/core/network/api_client.dart`

Static HTTP client wrapping the `http` package with automatic JWT auth header injection.

### Methods

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `get(path)` | URL path | `Future<dynamic>` | GET request |
| `post(path, {body})` | URL path, optional JSON body | `Future<dynamic>` | POST request |
| `put(path, {body})` | URL path, optional JSON body | `Future<dynamic>` | PUT request |
| `delete(path)` | URL path | `Future<dynamic>` | DELETE request |

### Request Flow
1. Build headers: `Content-Type: application/json` + `Authorization: Bearer <token>` if available
2. Construct full URL: `${AppConfig.apiBaseUrl}$path`
3. Execute request
4. Parse JSON response
5. If status code is 2xx: return parsed body
6. Otherwise: throw `ApiException` with status code and error message

### ApiException
```dart
class ApiException implements Exception {
  final int statusCode;
  final String message;
}
```

---

## Network - WebSocket Manager (`ws_manager.dart`)

**File:** `lib/core/network/ws_manager.dart`

Singleton WebSocket manager with automatic reconnection, offline queuing, and typed event streams.

### Architecture

```
WsManager (singleton)
  ├── WebSocketChannel (single connection)
  ├── Offline Message Queue
  ├── Connectivity Listener
  ├── Keepalive Ping Timer (25s interval)
  └── 5 Broadcast StreamControllers:
      ├── onGenerationComplete  → AI finished generating narrative
      ├── onMemoriesCurated     → New memories were created
      ├── onError               → Generation failure or server error
      ├── onConnectionState     → Connection status (bool)
      └── onInstanceLoaded      → Full instance state response
```

### Connection Lifecycle

1. **Connect:** `connect(token)` — authenticates via query param `?token=<jwt>`
2. **Ping:** Every 25 seconds, sends `{"action": "ping"}`
3. **Disconnect:** Triggers reconnect schedule
4. **Reconnect:** Exponential backoff (2s × attempt, max 30s, up to 10 attempts)
5. **Network Change:** `connectivity_plus` listener triggers reconnect on network restoration

### Message Routing

Incoming messages are routed by `type` field:
```dart
switch (msg['type']) {
  case 'generation_complete': → _generationCompleteController.add(msg)
  case 'memories_curated':    → _memoriesCuratedController.add(msg)
  case 'generation_failed':
  case 'error':               → _errorController.add(msg)
  case 'instance_loaded':     → _instanceLoadedController.add(msg)
  case 'pong': case 'ack': case 'connected': → (no-op)
}
```

### Offline Queue
If the connection is down when `send()` is called, messages are appended to `_offlineQueue`. When reconnection succeeds, `_flushOfflineQueue()` sends all queued messages.

### Public Methods

| Method | Parameters | Description |
|--------|-----------|-------------|
| `connect(token)` | JWT token string | Opens connection, starts connectivity listener |
| `send(message)` | `Map<String, dynamic>` | Sends JSON message or queues offline |
| `sendChatMessage(instanceId, message)` | Instance ID, text | Sends `{"action": "chat", ...}` |
| `loadInstance(instanceId)` | Instance ID | Sends `{"action": "load_instance", ...}` |
| `disconnect()` | — | Closes connection, cancels timers |
| `dispose()` | — | Closes all StreamControllers |

### Streams

| Stream | Type | Description |
|--------|------|-------------|
| `onConnectionState` | `Stream<bool>` | True when connected |
| `onGenerationComplete` | `Stream<Map>` | AI response with narrative + state diff |
| `onInstanceLoaded` | `Stream<Map>` | Full instance data on load |
| `onMemoriesCurated` | `Stream<Map>` | Newly curated memories |
| `onError` | `Stream<Map>` | Error messages |
| `isConnected` | `bool` getter | Current connection status |

---

## Storage - Secure Storage (`secure_storage.dart`)

**File:** `lib/core/storage/secure_storage.dart`

Wrapper around `flutter_secure_storage` for encrypted key-value storage.

### Keys

| Key | Constant | Content |
|-----|----------|---------|
| `auth_token` | `_tokenKey` | JWT authentication token |
| `user_data` | `_userKey` | JSON-encoded `User` object |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `saveToken(token)` | `Future<void>` | Stores JWT token |
| `getToken()` | `Future<String?>` | Retrieves JWT token |
| `saveUserData(json)` | `Future<void>` | Stores user JSON |
| `getUserData()` | `Future<String?>` | Retrieves user JSON |
| `clearAll()` | `Future<void>` | Deletes all stored data (logout) |

### Platform Behavior
- **iOS:** Keychain storage
- **Android:** Encrypted SharedPreferences (requires API 23+)
- **Web:** Not supported (will throw)

---

## Storage - Local DB (`local_db.dart`)

**File:** `lib/core/storage/local_db.dart`

SQLite cache using `sqflite` for offline event storage. Database file: `everlore_cache.db` (version 1).

### Tables

#### `events`
| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PK | Event UUID |
| `instance_id` | TEXT NOT NULL | Associated world instance |
| `sequence` | INTEGER NOT NULL | Event order number |
| `type` | TEXT NOT NULL | `narration`, `player_action`, etc. |
| `player_input` | TEXT | Player's message (nullable) |
| `ai_response` | TEXT | AI narrative text (nullable) |
| `scene_tag` | TEXT | Scene classification tag |
| `state_mutations` | TEXT | JSON state changes (unused in queries) |
| `flag_mutations` | TEXT | JSON flag changes (unused in queries) |
| `created_at` | TEXT NOT NULL | ISO 8601 timestamp |
| `is_optimistic` | INTEGER | 0 or 1; marks provisional events |

**Index:** `idx_events_instance` on `(instance_id, sequence)`

#### `instances`
| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PK | Instance UUID |
| `template_id` | TEXT NOT NULL | Source template |
| `title` | TEXT | Display name |
| `world_state` | TEXT | JSON state values |
| `active_flags` | TEXT | JSON flags |
| `last_active_at` | TEXT | Last activity timestamp |

#### `memories`
| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PK | Memory UUID |
| `instance_id` | TEXT NOT NULL | Associated instance |
| `text` | TEXT NOT NULL | Memory content |
| `type` | TEXT | Memory classification |
| `importance` | INTEGER | 1-5 importance rating |

**Index:** `idx_memories_instance` on `(instance_id)`

### Methods

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `getEvents(instanceId, {limit})` | Instance ID, default limit 50 | `Future<List<GameEvent>>` | Returns events ordered by sequence (ascending) |
| `insertEvent(event)` | `GameEvent` | `Future<void>` | Upserts event (REPLACE conflict algorithm) |
| `clearOptimisticEvents(instanceId)` | Instance ID | `Future<void>` | Removes all optimistic events for an instance |
| `clearInstanceCache(instanceId)` | Instance ID | `Future<void>` | Removes all events + memories for an instance |

### Conversion Methods
- `GameEvent.fromSqlite(row)` — creates `GameEvent` from database row map
- `GameEvent.toSqlite()` — converts `GameEvent` to row map for insertion
