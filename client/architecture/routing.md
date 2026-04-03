# Routing & Navigation

Everlore uses **`go_router`** for declarative, type-safe routing. All routes are defined in `lib/app_routes.dart`.

---

## Route Table

| Name | Path | Screen | Parameters |
|------|------|--------|------------|
| `home` | `/` | `HomeScreen` | — |
| `auth` | `/auth` | `AuthScreen` | — |
| `play` | `/play/:instanceId` | `PlayScreen` | `instanceId` (path) |
| `chronicle` | `/chronicle/:instanceId` | `ChronicleScreen` | `instanceId` (path) |
| `templates` | `/templates` | `BrowseTemplatesScreen` | — |
| `template_detail` | `/templates/:templateId` | `TemplateDetailScreen` | `templateId` (path) |

---

## Navigation API

### Push (adds to stack)
```dart
context.push('/play/${instance.id}');
context.push('/chronicle/${instanceId}');
context.push('/templates');
context.push('/templates/${template.id}');
```

### Push by name
```dart
context.pushNamed('auth');
```

### Pop (go back)
```dart
context.pop();
```

### Go (replace stack)
```dart
context.go('/play/${instance.id}');  // Used after creating instance from template
```

---

## Route Configuration

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      name: 'home',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/play/:instanceId',
      name: 'play',
      builder: (context, state) => PlayScreen(
        instanceId: state.pathParameters['instanceId']!,
      ),
    ),
    // ... other routes
  ],
);
```

The router is passed to `MaterialApp.router()` in `main.dart`:

```dart
MaterialApp.router(
  title: 'Everlore',
  debugShowCheckedModeBanner: false,
  theme: NexusTheme.dark,
  routerConfig: router,
);
```

---

## Typical Navigation Flows

### New Player Flow
```
/ → /auth → (Google Sign-In or Phone OTP) → / → /templates → (tap template) → /templates/:id → (Enter This World) → /play/:instanceId
```

### Return Player Flow
```
/ → (existing secure session) → (tap world card) → /play/:instanceId
```

### Chronicle Access
```
/play/:instanceId → (Chronicle icon) → /chronicle/:instanceId
```

### Auth Notes

- The router still has no global guard; authentication is enforced by the backend.
- The home screen now surfaces unauthorized API responses by linking users to `/auth`.
- `AuthScreen` handles both Google Sign-In and phone OTP, then redirects with `context.go('/')` after a backend JWT is stored.

---

## Adding a New Route

1. Add a `GoRoute` entry in `app_routes.dart`
2. Import the screen widget
3. Optionally use `state.pathParameters` for dynamic segments
4. Navigate using `context.push()` or `context.go()`
