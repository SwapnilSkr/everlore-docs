# Shared Widgets

Reusable UI components shared across multiple features.

---

## ErrorBanner

**File:** `lib/shared/widgets/error_banner.dart`

A horizontal error bar with icon, message, and optional retry button.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `message` | `String` | Error text to display |
| `onRetry` | `VoidCallback?` | Optional retry action |

### Styling
- Background: `Colors.red` with 15% opacity
- Icon: `Icons.error_outline` in `Colors.redAccent`
- Text: `Colors.redAccent`, 13px

### Usage
```dart
if (state.error != null)
  ErrorBanner(
    message: state.error!,
    onRetry: () => cubit.clearError(),
  )
```

---

## StatBar

**File:** `lib/shared/widgets/stat_bar.dart`

A labeled progress bar showing a numeric stat value relative to its maximum.

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `label` | `String` | required | Stat name (e.g., "Health") |
| `value` | `num` | required | Current value |
| `max` | `num` | `100` | Maximum value |
| `color` | `Color?` | `null` | Override bar color |

### Color Logic
If no `color` is provided, the bar color is determined by the value fraction:

| Fraction Range | Color |
|---------------|-------|
| > 0.7 | `Colors.redAccent` (high/danger) |
| > 0.4 | `Colors.amberAccent` (medium) |
| <= 0.4 | `Colors.greenAccent` (low/safe) |

### Display
```
Health                 80/100
████████████████░░░░░░░░░░░░   ← 6px rounded progress bar
```

### Usage
```dart
StatBar(label: 'Health', value: 80, max: 100)
StatBar(label: 'Mana', value: 40)  // max defaults to 100
```

Used in:
- `WorldStateBar` (play feature) — shows live world stats
- `TemplateDetailScreen` (templates feature) — shows stat definitions preview

---

## LoadingShimmer

**File:** `lib/shared/widgets/loading_shimmer.dart`

An animated shimmer effect placeholder for loading states.

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `width` | `double` | `double.infinity` | Shimmer width |
| `height` | `double` | `16` | Shimmer height |

### Animation
- Uses `AnimationController` with 1500ms duration, repeating
- Gradient sweeps from `Color(0xFF1a1a2e)` → `Color(0xFF2a2a3e)` → `Color(0xFF1a1a2e)`
- Gradient position animates horizontally using `Alignment(-1.0 + 2.0 * value, 0)`

### Usage
```dart
LoadingShimmer(width: 200, height: 12)
```

> **Note:** This widget is defined but not currently used in the app. It's available for future loading states.

---

## Widget Dependency Map

```
ErrorBanner ← PlayScreen
StatBar     ← WorldStateBar, TemplateDetailScreen
LoadingShimmer ← (unused, available)
```
