# Routing & Navigation

**File:** `lib/app_routes.dart` — `go_router` with four shell branches + root overlay routes.  
**Nav UI:** `lib/shared/widgets/everlore_nav_bar.dart` — `ScaffoldWithNavBar` + floating glass `EverloreNavBar`.

---

## Route table

### Pre-app (no bottom nav)

| Path | Screen |
|------|--------|
| `/splash` | SplashScreen |
| `/welcome` | WelcomeScreen |
| `/auth` | AuthScreen |
| `/onboarding` | OnboardingScreen |

### Shell tabs (`StatefulShellRoute.indexedStack` → `ScaffoldWithNavBar`)

Four primary branches. Each keeps its own stack/state across tab switches. Labels on the bar: **Explore · Realms · Worlds · Personas**, plus a center **Create** action (not a branch).

| Branch index | Path | Screen | Nav label |
|--------------|------|--------|-----------|
| 0 | `/discover` | DiscoverScreen | Explore |
| 1 | `/` | HomeScreen (grouped playthroughs) | Realms |
| 2 | `/my-worlds` | MyWorldsScreen | Worlds |
| 3 | `/personas` | PersonasScreen | Personas |

**Create chooser** (center brass `+`): bottom sheet → `context.push('/my-worlds/forge')` or `context.push('/characters/new')`.

### Over shell (root navigator — covers nav bar)

| Path | Screen |
|------|--------|
| `/templates` | BrowseTemplatesScreen |
| `/templates/:templateId` | TemplateDetailScreen |
| `/realms/:templateId` | RealmPlaythroughsScreen |
| `/play/:instanceId` | **PlayScreen** |
| `/chronicle/:instanceId` | **ChronicleScreen** (`?section=` optional initial tab) |
| `/realm/:instanceId` | **RealmScreen** (calm map of an open playthrough; `extra: RealmScreenArgs`) |
| `/characters/new` | CreateCharacterScreen |
| `/my-worlds/forge` | ForgeWorldRoute (new world) |
| `/my-worlds/:templateId/forge` | ForgeWorldRoute (edit; `extra: WorldTemplate?`) |
| `/profile` | AuthScreen (profile / account) |
| `/membership` | **BillingScreen** (Membership & Ink) |

---

## Common navigation

```dart
// Enter play
context.push('/play/$instanceId');

// Open Lore Tome from play menu
context.push('/chronicle/$instanceId');

// Realm overview (from play header)
context.push('/realm/$instanceId', extra: RealmScreenArgs(...));

// Membership & Ink
context.push('/membership');

// Forge
context.push('/my-worlds/forge');
context.push('/characters/new');

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
  Realms tab (/) → tap story → play

Create:
  nav + → Forge a World | Create a Character

Chronicle:
  play ⋮ menu → chronicle
  OR play "older history" link

World Actions (in play composer when empty):
  continue / time skip / travel / set kinship / review relation candidates

Membership:
  profile/auth → /membership

Side chat:
  chronicle → Bonds tab → 💬 icon (not from play bond sheet)
```

---

## Auth

No global router guard — backend returns 401; splash/auth flow handles session. JWT in `SecureStore`; `AuthService.sessionEpoch` notifies widgets to refresh feeds.

---

## Theme

`MaterialApp.router` uses `EverloreTheme.dark` from `app/theme/nexus_theme.dart`.
