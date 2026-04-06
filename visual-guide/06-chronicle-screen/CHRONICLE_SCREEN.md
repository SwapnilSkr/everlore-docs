# 06 — Chronicle Screen (Lore Tome)

> **Route:** `/chronicle/:instanceId` | **File:** `lib/features/chronicle/presentation/chronicle_screen.dart` | **Auth:** Required

---

## Purpose

A read-only historical view of the game's events and AI-generated memories. Two tabs: **Timeline** (full event log) and **Echoes** (curated memories). Users can edit or delete memories.

---

## Layout

```
┌──────────────────────────────────────────┐
│  ←  📖 Lore Tome                     │  ← Header: void0 bg, white10 bottom border
│                                        │
│  ┌ Timeline ──┐  ┌ Echoes ──────┐     │  ← Tab buttons
│  │ ○ active  │  │  ○ inactive │     │
│  └───────────┘  └─────────────┘     │
├──────────────────────────────────────────┘

=== Timeline Tab ===

┌──────────────────────────────────────────┐
│  (Same NarrativeBubble list as PlayScreen)      │
│  ┌─ Narrator Bubble ──────────────────┐ │
│  │  ◉ NARRATOR                       │ │
│  │  [Markdown content]                 │ │
│  │  [Scene tag badge]                  │ │
│  └─────────────────────────────────────┘ │
│  ┌─ Player Bubble ────────────────────┐ │
│  │  [Player text, right-aligned]        │ │
│  └─────────────────────────────────────┘ │
└──────────────────────────────────────────┘

=== Echoes Tab ===

┌──────────────────────────────────────────┐
│  ┌─ MemoryCard ─────────────────────┐   │
│  │  [Type Badge] ★★★☆☆  [✏] [🗑]    │   │
│  │  Memory text content…              │   │
│  │  (Faded — no longer active)         │   │
│  └────────────────────────────────────┘   │
│                                        │
│  ┌─ MemoryCard ─────────────────────┐   │
│  │  [Bond] ★★★★★       [✏] [🗑]    │   │
│  │  The blacksmith swore an oath…     │   │
│  └────────────────────────────────────┘   │
└──────────────────────────────────────────┘

=== Edit Dialog (on memory edit) ===

┌──────────────────────────────────────────┐
│  🔖 Edit Echo                        ✕ │  ← Dialog, void2, 20px radius
│                                        │
│  MEMORY                                │  ← sectionHeader
│  [Text field, 4 lines, void3 fill]     │
│                                        │
│  TYPE                                  │
│  ○ Bond  ○ Oath  ○ Lore  ○ Sight     │  ← Pill selector with type colors
│  ○ Feeling  ○ Secret                  │
│                                        │
│  IMPORTANCE          ★★★★☆            │  ← Star rating (1-5)
│                                        │
│  [Cancel]              [Save Echo]      │  ← OutlinedButton + ElevatedButton
└──────────────────────────────────────────┘
```

---

## Tab Switching

| Tab | Icon | State |
|---|---|---|
| Timeline | `Icons.timeline` | `ChronicleTab.timeline` |
| Echoes | `Icons.bookmark_outline` | `ChronicleTab.memories` |

Active tab: gold @ 12% bg, goldDim @ 60% border, gold text/icon
Inactive tab: void2 bg, goldDim @ 15% border, ash text/icon

---

## MemoryCard Component

### Type Badge

| Type | Label | Icon | Color |
|---|---|---|---|
| relationship | Bond | favorite_outline | `#EC4899` |
| promise | Oath | handshake_outlined | `#D4A843` (gold) |
| lore | Lore | auto_stories | `#22D3EE` (cyanBright) |
| observation | Sight | visibility_outlined | `#A855F7` (violetBright) |
| emotion | Feeling | sentiment_satisfied_outlined | `#F97316` |
| secret | Secret | lock_outline | `#DC2626` (crimson) |

### Importance Stars
- 5 stars total
- Filled: `Icons.star_rounded`, gold
- Empty: `Icons.star_outline_rounded`, white20

### Actions
- Edit (`Icons.edit_outlined`, ash) → Opens EditMemoryDialog
- Delete (`Icons.delete_outline`, crimson @ 60%) → Confirm dialog "Erase This Echo?"

---

## Empty States

| Tab | Icon | Title | Subtitle |
|---|---|---|---|
| Timeline | `history_edu` | "No story yet" | "Your adventures will be recorded here." |
| Echoes | `bookmark_border` | "No echoes yet" | "Memories from your journey will appear here." |

Loading: "Unrolling the scroll..." (italic, ash)

---

## State

- `ChronicleCubit` → `ChronicleState`
- Fields: `instanceId`, `events`, `memories`, `activeTab`, `isLoading`
- Actions: `loadEvents()`, `switchTab()`, `editMemory()`, `deleteMemory()`
