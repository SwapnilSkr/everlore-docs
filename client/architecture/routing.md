# Routing & Navigation

**File:** `lib/app_routes.dart` — `go_router` with shell branches + root overlay routes.

---

## Route table

### Pre-app (no bottom nav)

| Path | Screen |
|------|--------|
| `/splash` | SplashScreen |
| `/welcome` | WelcomeScreen |
| `/auth` | AuthScreen |
| `/onboarding` | OnboardingScreen |

### Shell tabs (`ScaffoldWithNavBar`)

| Branch | Path | Screen |
|--------|------|--------|
| Explore | `/discover` | DiscoverScreen |
| Realms | `/realms` | HomeScreen (grouped playthroughs) |
| Profile | `/profile` | PersonasScreen / profile |

### Over shell (root navigator — covers nav bar)

| Path | Screen |
|------|--------|
| `/templates` | BrowseTemplatesScreen |
| `/templates/:templateId` | TemplateDetailScreen |
| `/realms/:templateId` | RealmPlaythroughsScreen |
| `/play/:instanceId` | **PlayScreen** |
| `/chronicle/:instanceId` | **ChronicleScreen** |
| `/forge/world` | ForgeWorldRoute |
| `/forge/character` | CreateCharacterScreen |
| `/my-worlds` | MyWorldsScreen |

---

## Common navigation

```dart
// Enter play
context.push('/play/$instanceId');

// Open Lore Tome from play menu
context.push('/chronicle/$instanceId');

// After creating instance
context.go('/play/$newInstanceId');

// Back
context.pop();
```

---

## Typical flows

```text
New player:
  splash → welcome → auth → onboarding → discover
  → template detail → realm picker → play

Return player:
  realms tab → tap story → play

Chronicle:
  play ⋮ menu → chronicle
  OR play "older history" link

Side chat:
  chronicle → bonds tab → 💬 icon (not from play bond sheet)
```

---

## Auth

No global router guard — backend returns 401; splash/auth flow handles session. JWT in `SecureStore`; `AuthService.sessionEpoch` notifies widgets to refresh feeds.

---

## Theme

`MaterialApp.router` uses `EverloreTheme.dark` from `app/theme/nexus_theme.dart`.
