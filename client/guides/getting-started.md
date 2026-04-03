# Getting Started — Developer Guide

## Prerequisites

- Flutter SDK 3.x (Dart SDK ^3.8.1)
- Android Studio / VS Code with Flutter extensions
- iOS: Xcode 15+ (for iOS/macOS builds)
- Android: Android SDK with API 23+ (for `flutter_secure_storage`)
- A running Everlore backend server (default: `http://localhost:3000`)

---

## Setup

### 1. Install Dependencies
```bash
cd everlore
flutter pub get
```

### 2. Configure Environment
Create a `.env` file in the `everlore/` root:
```
API_BASE_URL=http://localhost:3000
WS_BASE_URL=ws://localhost:3000
GOOGLE_WEB_CLIENT_ID=your-google-web-client-id.apps.googleusercontent.com
```

`GOOGLE_WEB_CLIENT_ID` should be the same Web OAuth client ID that the server uses as `GOOGLE_CLIENT_ID`.

### 3. Configure Backend URL (Optional)
For local development, the defaults work:
- API: `http://localhost:3000`
- WebSocket: `ws://localhost:3000`

For staging/production:
```bash
flutter run --dart-define=API_BASE_URL=https://api.everlore.io --dart-define=WS_BASE_URL=wss://api.everlore.io
```

### 4. Run the App
```bash
flutter run
```

---

## Project Conventions

### File Naming
- Files: `snake_case.dart` (e.g., `home_cubit.dart`, `world_card.dart`)
- Classes: `PascalCase` (e.g., `HomeCubit`, `WorldCard`)
- Private classes: prefix with `_` (e.g., `_HomeView`, `_PlayViewState`)

### State Management Pattern
Every feature follows:
```
features/<name>/
├── data/<name>_repository.dart    ← static methods, API calls
├── presentation/
│   ├── <name>_screen.dart         ← BlocProvider + main widget
│   └── widgets/                   ← feature-specific components
└── state/<name>_cubit.dart        ← Cubit + Equatable state
```

### Import Paths
Always use package-relative imports:
```dart
import 'package:everlore/core/network/api_client.dart';
import 'package:everlore/shared/models/user.dart';
```

### Color Constants
- Theme colors go through `NexusTheme.dark` via `Theme.of(context)`
- Feature-specific colors are hardcoded in widget files
- Named color constants (e.g., `Color(0xFF0d0d1a)`) are repeated across files

### No Code Comments
The codebase follows a no-comment convention. Code should be self-documenting through clear naming.

---

## Common Tasks

### Adding a New Screen

1. Create the screen file: `lib/features/<name>/presentation/<name>_screen.dart`
2. Add a route in `lib/app_routes.dart`:
   ```dart
   GoRoute(
     path: '/<name>',
     name: '<name>',
     builder: (context, state) => const <Name>Screen(),
   )
   ```
3. Navigate to it: `context.push('/<name>')`

### Adding a New API Endpoint

1. Add a static method to the relevant repository:
   ```dart
   static Future<Result> doSomething() async {
     final response = await ApiClient.get('/endpoint');
     return Result.fromJson(response);
   }
   ```
2. Call it from the feature's Cubit:
   ```dart
   Future<void> doSomething() async {
     try {
       final result = await Repository.doSomething();
       emit(state.copyWith(...));
     } catch (e) {
       emit(state.copyWith(error: e.toString()));
     }
   }
   ```

### Adding a New WebSocket Message Type

1. Add a `StreamController` in `WsManager`:
   ```dart
   final _myEventController = StreamController<Map<String, dynamic>>.broadcast();
   Stream<Map<String, dynamic>> get onMyEvent => _myEventController.stream;
   ```
2. Handle the message type in `_onMessage()`:
   ```dart
   case 'my_event_type':
     _myEventController.add(msg);
     break;
   ```
3. Subscribe in the relevant Cubit's `_init()`:
   ```dart
   _mySub = _ws.onMyEvent.listen((msg) { ... });
   ```
4. Cancel in `close()`: `_mySub.cancel();`
5. Dispose in `WsManager.dispose()`: `_myEventController.close();`

### Adding a New Data Model

1. Create `lib/shared/models/<model>.dart`
2. Define the class with `const` constructor
3. Add `fromJson(Map<String, dynamic> json)` factory constructor
4. Handle both `id` and `_id` keys from the API
5. Use in repository layer to deserialize API responses

---

## Testing

Run tests:
```bash
flutter test
```

The `test/` directory contains the default Flutter counter app test. Tests for Everlore features should be added under `test/features/<name>/`.

### Suggested Test Structure
```
test/
├── core/
│   ├── auth_service_test.dart
│   ├── api_client_test.dart
│   └── ws_manager_test.dart
├── features/
│   ├── home/
│   │   ├── home_cubit_test.dart
│   │   └── home_repository_test.dart
│   ├── play/
│   │   └── play_cubit_test.dart
│   ├── chronicle/
│   │   └── chronicle_cubit_test.dart
│   └── templates/
│       └── template_repository_test.dart
└── shared/
    ├── models_test.dart
    └── widgets_test.dart
```

---

## Build & Release

### Android APK
```bash
flutter build apk --release
```

### iOS IPA
```bash
flutter build ios --release
```

### Web
```bash
flutter build web --release
```

---

## Known Issues & TODOs

- The `lib/screens/` directory still contains shared entry screens; `auth_screen.dart` is now the active auth UI for Google Sign-In and phone OTP, while `home_screen.dart` remains placeholder/legacy
- `LoadingShimmer` widget is defined but unused
- Error states in `HomeCubit` are only partially surfaced in the UI
- `context.go()` in `TemplateDetailScreen` replaces the entire navigation stack (user cannot go back to templates)
- No global authentication guards on routes — access control is enforced by the backend and unauthorized states redirect users toward `/auth`
- The `UserPreferences.narrationLength` and `preferredModel` fields exist in the model but are not used in the UI
- WebSocket does not handle token refresh — if the JWT expires during a session, the connection will fail silently

### Authentication Setup Notes

- Google Sign-In requires platform setup outside this repo, including the correct Web OAuth client ID and any Android SHA-1 / iOS bundle registration needed by Google.
- Phone OTP is backend-driven; the Flutter client only calls `/auth/otp/send` and `/auth/otp/verify`.
- For local testing with the server in mock Twilio mode, the accepted OTP is `123456`.
