# Theme & Styling

Everlore uses a single dark theme called **NexusTheme** defined in `lib/app/theme/nexus_theme.dart`.

---

## Color Palette

### Background Colors
| Name | Color | Usage |
|------|-------|-------|
| Deep Navy | `#0d0d1a` | Scaffold background, AppBar background |
| Dark Surface | `#1a1a2e` | Cards, input fields, AI response bubbles |
| Mid Surface | `#12122a` | World state bar background |
| Light Surface | `#2a2a3e` | Dropdown backgrounds, shimmer highlight |
| Surface Opacity | `Colors.white10` | Chip backgrounds, borders |

### Accent Colors
| Name | Color | Usage |
|------|-------|-------|
| Primary | `Colors.purpleAccent` | Buttons, FABs, active tabs, loading indicators |
| Secondary | `Colors.cyanAccent` | ColorScheme secondary |
| Error | `Colors.redAccent` | Error states, delete actions |

### Text Colors
| Name | Color | Usage |
|------|-------|-------|
| White | `Colors.white` | Titles, primary text |
| White 70% | `Colors.white70` | Body text, AI narrative |
| White 54% | `Colors.white54` | Secondary text, labels |
| White 38% | `Colors.white38` | Tertiary text, hints, timestamps |
| White 30% | `Colors.white30` | Input hints |
| White 24% | `Colors.white24` | Icons in disabled state |
| White 12% | `Colors.white12` | Disabled FAB background |

---

## Theme Data

```dart
ThemeData(
  brightness: Brightness.dark,
  scaffoldBackgroundColor: Color(0xFF0d0d1a),
  primaryColor: Colors.purpleAccent,
  colorScheme: ColorScheme.dark(
    primary: Colors.purpleAccent,
    secondary: Colors.cyanAccent,
    surface: Color(0xFF1a1a2e),
    error: Colors.redAccent,
  ),
)
```

### Component Themes

#### AppBarTheme
- Background: `#0d0d1a` (matches scaffold)
- No elevation (flat)
- Title: white, 18px, semi-bold
- Icons: white 70%

#### CardTheme
- Background: `#1a1a2e`
- No elevation
- 12px border radius

#### ElevatedButtonTheme
- Background: `Colors.purpleAccent`
- Foreground: `Colors.white`
- 12px border radius

#### InputDecorationTheme
- Filled with `#1a1a2e` background
- No visible border (BorderSide.none)
- Hint text: white 30%

#### TextTheme
| Style | Color |
|-------|-------|
| `bodyLarge` | White |
| `bodyMedium` | White 70% |
| `bodySmall` | White 54% |

---

## Hardcoded Colors

Some widgets use hardcoded colors instead of theme references:

| Widget | Color | Reason |
|--------|-------|--------|
| `NarrativeBubble` player bubble | `Colors.purpleAccent.withOpacity(0.2)` + border | Distinct from theme purple |
| `NarrativeBubble` AI bubble | `Color(0xFF1a1a2e)` | Matches theme surface |
| `PlayerInput` container | `Color(0xFF0d0d1a)` | Matches scaffold |
| `WorldStateBar` | `Color(0xFF12122a)` | Subtle distinction from surface |
| `MemoryCard` | `Color(0xFF1a1a2e)` | Matches card theme |
| `EditMemoryDialog` | `Color(0xFF1a1a2e)` + `Color(0xFF2a2a3e)` | Dark dialog styling |

---

## Adding Theme Support

The theme is currently dark-only. To add light mode support:

1. Add a `NexusTheme.light` getter with light color equivalents
2. Store the user's preference in `UserPreferences.theme`
3. Switch `theme` in `MaterialApp.router()` based on the preference
