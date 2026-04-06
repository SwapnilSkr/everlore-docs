# Shared Widgets & Reusable Components

> **Directory:** `lib/shared/widgets/` | **Theme:** `lib/app/theme/nexus_theme.dart`

---

## 1. ErrorBanner

> **File:** `lib/shared/widgets/error_banner.dart`

### Purpose
A dismissable error banner with optional retry button.

### Layout
```
┌──────────────────────────────────────────┐
│  ⚠ Error message text        [Retry]    │  ← crimson @10% bg, @30% border
└──────────────────────────────────────────┘
```

### Specs
| Property | Value |
|---|---|
| Margin | 16px horizontal, 6px vertical |
| Padding | 14/10/10/10 |
| Background | crimson @10% |
| Border | 1px crimson @30% |
| Radius | 10px |
| Icon | error_outline, 16px, crimson |
| Text | 13px, crimson |
| Retry button | TextButton, crimson, 12px |

---

## 2. LoadingShimmer

> **File:** `lib/shared/widgets/loading_shimmer.dart`

### Purpose
An animated shimmer placeholder for loading states.

### Layout
```
┌──────────────────────────────────┐
│  ░░░░▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░  │  ← Animated sweep gradient
└──────────────────────────────────┘
```

### Specs
| Property | Value |
|---|---|
| Default width | double.infinity |
| Default height | 16px |
| Radius | 4px |
| Gradient sweep | void2 → void4 → void2 |
| Animation | 1500ms, repeat |
| Sweep direction | Left to right (-1.0 → 1.0 offset) |

---

## 3. StatBar

> **File:** `lib/shared/widgets/stat_bar.dart`

### Purpose
A reusable RPG-style stat bar with label, value, and gradient fill.

### Layout
```
          Label              50 / 100
          ████████████░░░░░░░░░░░░░
```

### Specs
| Property | Value |
|---|---|
| Padding | 4px vertical |
| Label | 12px, ash |
| Value | 11px, w600, dynamic color |
| Track | 5px, void4, radius 3px |
| Fill | 5px, gradient (color@70% → color), radius 3px |
| Shadow | color @35%, blur 4px |
| Default max | 100 |

### Color Logic
| Percentage | Color |
|---|---|
| ≥ 60% | verdant |
| ≥ 30% | ember |
| < 30% | crimson |

---

## 4. NarrativeBubble (Play & Chronicle)

> **File:** `lib/features/play/presentation/widgets/narrative_bubble.dart`

### Purpose
Displays a single game event with player input (right-aligned) and AI narration (left-aligned).

### Layout
```
                          ┌─────────────────────────────┐
                          │ Player action text          │  ← Right-aligned, violet gradient
                          └─────────────────────────────┘
┌─────────────────────────────────────┐
│ ◉ NARRATOR                          │  ← Left-aligned, void2 bg
│                                     │
│ Narrative text with Markdown...     │  ← MarkdownBody with custom styling
│                                     │
│                [COMBAT]              │  ← Scene tag pill badge
└─────────────────────────────────────┘
```

### Player Bubble
| Property | Value |
|---|---|
| Alignment | centerRight |
| Margin | 64/8/16/4 |
| Padding | 16px horizontal, 12px vertical |
| Background | Gradient: violetDim@80% → violet@40% |
| Border | violetBright @20% |
| Radius | topLeft:16, topRight:4, bottomLeft:16, bottomRight:16 |
| Text | 15px, parchment, height 1.5 |

### Narrator Bubble
| Property | Value |
|---|---|
| Margin | 16/4/64/8 |
| Padding | 16px all |
| Background | void2 |
| Border | goldDim @18% |
| Radius | topLeft:4, topRight:16, bottomLeft:16, bottomRight:16 |
| NARRATOR label | gold @50%, 9px, w700, letterSpacing 1.5 |
| Text | MarkdownBody with custom styleSheet |

### Scene Tag Badge
| Property | Value |
|---|---|
| Padding | 8/3 |
| Radius | 20px |
| Background | color @10% |
| Border | color @30% |
| Text | 9px, w700, letterSpacing 1.2, uppercase |

### Scene Tag Colors
| Tag | Color |
|---|---|
| combat | crimson |
| intimate | #EC4899 |
| exploration | verdant |
| existential | cyanBright |
| cosmic | violetBright |
| dialogue | gold |
| mundane | ash |

### Generating Indicator
- Pulsing animation: 1200ms, easeInOut, repeat reverse
- Gold icon + "The world is weaving your tale…" (italic, pulsing opacity)

---

## 5. PlayerInput (Play)

> **File:** `lib/features/play/presentation/widgets/player_input.dart`

### Layout
```
┌──────────────────────────────────────────┐
│ void0 bg, white10 top border              │
│                                          │
│  ┌──────────────────────────────┐  ◉    │  ← Text field + send button
│  │ What do you do?              │  │    │
│  └──────────────────────────────┘  │    │
│                                    │    │
└──────────────────────────────────────────┘
```

### Input Container
| Property | Value |
|---|---|
| Background | void2 |
| Radius | 18px |
| Border (unfocused) | goldDim @20% |
| Border (focused) | goldDim @60% |
| Text | 15px, parchment, height 1.4 |
| Hint | ash @40%, 14px, italic |
| Max lines | 5 (expanding) |
| Min lines | 1 |

### Send Button
| Property | Value |
|---|---|
| Size | 44×44 circle |
| Active | gold bg, void0 icon, gold shadow |
| Disabled | void3 bg, ash @30% icon |
| Generating | void3 bg, gold progress indicator |
| Icon | send_rounded, 18px |

### Hint Texts
| State | Text |
|---|---|
| Generating | "The story unfolds…" |
| Disconnected | "Reconnecting to the realm…" |
| Normal | "What do you do?" |

---

## 6. WorldStateBar (Play)

> **File:** `lib/features/play/presentation/widgets/world_state_bar.dart`

### Collapsed State
```
┌──────────────────────────────────────────────┐
│  🛡 REALM STATUS   ● ● ● ●         [▼]     │  ← Mini stat dots
└──────────────────────────────────────────────┘
```

### Expanded State
```
┌──────────────────────────────────────────────┐
│  🛡 REALM STATUS                       [▲]   │
│                                              │
│  Health        ████████████░░░░░░       75    │  ← Full RPG stat bars
│  Mana          ████████░░░░░░░░░░       50    │
│  Sanity        ████████████████░░       85    │
└──────────────────────────────────────────────┘
```

### Specs
| Property | Value |
|---|---|
| Background | void0 |
| Border | white10 bottom |
| Toggle padding | 16px horizontal, 10px vertical |
| "REALM STATUS" | gold @70%, 9px, w700, letterSpacing 2 |
| Toggle icon | ash @50%, 16px |
| Stat label width | 90px, 11px, ash |
| Value width | 36px, right-aligned |
| Bar height | 6px, radius 3px |
| Animation | 300ms, easeInOut |

### Mini Stat Dots (Collapsed)
- 6×6 circles, colored by stat level
- Box shadow: color @40%, blur 4px
- Shows max 4 dots

---

## 7. MemoryCard (Chronicle)

> **File:** `lib/features/chronicle/presentation/widgets/memory_card.dart`

### Layout
```
┌──────────────────────────────────────────┐
│  [◉ Bond]  ★★★☆☆         [✏] [🗑]     │  ← Type badge + stars + actions
│                                          │
│  Memory text content goes here…          │  ← 14px, ash, height 1.6
│                                          │
│  Faded — no longer active                │  ← (if archived) 10px, italic
└──────────────────────────────────────────┘
```

### Specs
| Property | Value |
|---|---|
| Margin | 16/6/16/2 |
| Padding | 14px |
| Background | void2 |
| Radius | 14px |
| Border | typeColor @20% |
| Star icons | 12px, gold (filled) or white20 (outline) |
| Action icons | edit_outlined (ash), delete_outline (crimson @60%) |

---

## 8. EditMemoryDialog (Chronicle)

> **File:** `lib/features/chronicle/presentation/widgets/edit_dialog.dart`

### Layout
```
┌──────────────────────────────────────────┐
│  ◉ Edit Echo                       [✕]   │
│                                          │
│  MEMORY                                  │  ← sectionHeader
│  ┌──────────────────────────────────┐    │
│  │ Text field (4 lines)            │    │  ← void3 fill, goldDim border
│  └──────────────────────────────────┘    │
│                                          │
│  TYPE                                    │  ← sectionHeader
│  [◉ Bond] [Oath] [Lore] [Sight]…        │  ← Wrap of pill selectors
│                                          │
│  IMPORTANCE            ★★★☆☆            │  ← Tap-to-set star rating
│                                          │
│  [Cancel]          [Save Echo]           │  ← OutlinedButton + ElevatedButton
└──────────────────────────────────────────┘
```

### Type Selector Pills
- Selected: typeColor @15% bg, @60% border, typeColor text
- Unselected: void3 bg, goldDim @15% border, ash text
- 150ms animation

---

## 9. WorldCard (Home)

> **File:** `lib/features/home/presentation/widgets/world_card.dart`

### Layout
```
┌──────────────────────────────────────────┐
│  ╭──╮                                    │  ← Gradient background
│  │◉ │ Title of World           [⋯]      │  ← Hash-based accent color
│  ╰──╯ Sentient World                     │
│                                          │
│  Description text, max 2 lines…          │  ← 13px, ash
│  ────────────────────────────────────    │  ← Accent @10% divider
│  ◉ 42  ◉ 15  [combat]     3h ago        │  ← Stats + scene + time
└──────────────────────────────────────────┘
```

### Specs
| Property | Value |
|---|---|
| Margin | 20/8/20/4 |
| Padding | 20px |
| Radius | 18px |
| Background | LinearGradient (hash-based) |
| Border | accentColor @25% |
| Shadow | accentColor @6%, blur 24, offset 4 |
| Icon circle | 42×42, accent @12% bg, @30% border |
| Title | 17px, w700, parchment |
| Description | 13px, ash, maxLines 2 |
| Stats row | 13px icons + 12px text |
| Scene tag | pill, 10px |
| Time ago | ash @60%, 11px |

---

## 10. MyWorldCard (Creator)

> **File:** `lib/features/creator/presentation/widgets/my_world_card.dart`

### Layout
```
┌─── 3px top stripe (draft=ember / live=verdant) ────────┐
│  ╭──╮  Title             ┌────────┐                    │
│  │◉ │  Conscious Soul    │ DRAFT  │                    │  ← Status badge
│  ╰──╯  [18+]             └────────┘                    │
│                                                        │
│  Description text, max 2 lines…                        │
│  [📊 3 stats] [🏷 combat] [🏷 magic] [+2]             │  ← Info chips
│  Forged 3d ago                                         │
│  ──────────────────────────────────────────────         │
│  [✏ Edit]                    [🌐 Release to Realm]     │  ← (draft only)
└────────────────────────────────────────────────────────┘
```

### Specs
| Property | Value |
|---|---|
| Margin | 0 0 12px |
| Radius | 16px |
| Background | Gradient void2 → void1@95% |
| Top stripe | 3px, gradient (ember/verdant @60% → @20%) |
| Draft border | ember @30% |
| Published border | verdant @30% |
| Status badge | ember/verdant @12% bg, @40% border |
| Info chips | void4@50% bg, 10px text |
| Action text | 13px, w600 |
| Time ago | 11px, ash |
