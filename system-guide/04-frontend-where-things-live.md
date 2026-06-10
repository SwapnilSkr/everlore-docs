# Frontend — Where Things Live

Map of the Flutter app: screens, buttons, and when they appear.

---

## App structure (high level)

```text
┌─────────────────────────────────────────┐
│  Shell nav (Discover / Realms / Profile) │
└─────────────────────────────────────────┘
         │
         ├── Discover → browse worlds
         ├── Realms → your playthroughs
         └── Play → /play/:instanceId  ← main game
                    │
                    └── Chronicle → /chronicle/:instanceId  ← Lore Tome
```

---

## Play screen (the game)

**File:** `lib/features/play/presentation/play_screen.dart`

### What you always see (when loaded)

| UI | Location | Notes |
|----|----------|-------|
| Back | Top left | Exit play |
| Realm menu ⋮ | Top right | Chronicle, timeline, thoughts, settings |
| Connection dot | Header | Green = WebSocket live |
| Chat bubbles | Center scroll | Your messages + AI narration |
| Text composer | Bottom | Type or use prefilled chips |

### What appears conditionally

| UI | When it shows | When it hides |
|----|---------------|---------------|
| **World stat bar** | World has RPG stats | Stat-less character worlds |
| **Bond rail** | ≥1 side character with relationship meters | No bonded NPCs yet |
| **Choice chips** | Latest turn done, server sent choices | Generating, replaying, disconnected |
| **Advance time button** | Composer empty, connected | You typed text, generating |
| **"Older history" link** | More than 20 events exist | All history loaded |
| **Milestone toast** | Server unlocked a milestone | Auto-dismisses |
| **Protagonist onboarding** | GM world, no player character yet | After you name yourself |
| **Error bar** | Something failed | You dismiss |

### Bond rail (relationship presence)

**File:** `widgets/bond_rail.dart`

- Up to **5** characters shown as circular meters
- Color = dominant bond (trust green, affection rose, fear violet, rivalry amber)
- **Dimmed** if character not in current scene (when scene knows who's present)
- **Tap** → action sheet (approach, ask, seek out, "what they remember")

### Narration bubble extras

**File:** `widgets/narrative_bubble.dart`

| Element | When |
|---------|------|
| **Gold underlined names** | Settled AI text — tap for bond actions |
| **Dotted underlined places/things** | Settled AI text — tap for memory lens |
| **Travel header** | Event was a travel turn |
| **Time passage header** | Calendar tick / time skip |
| **Scene tag badge** | dialogue, romantic, combat, etc. |
| **Continue button** | Latest turn, you haven't acted |
| **Replay button** | Latest turn with AI text |
| **Replay arrows** | Multiple variants saved |

### Tapping a name or place (in-play memory lens)

- **No API call** — filters the last **50** memories already on device
- Shows memories that "concern" that name, by importance
- For full character history → Lore Tome → Bonds → tap character

### Realm menu (⋮)

| Menu item | Opens |
|-----------|-------|
| **Chronicle** | Lore Tome (7 tabs) |
| **Story Timeline** | Milestones sheet → link to Chronicle |
| **Thoughts** | Cast list, focus character, edit cards |
| **Settings** | POV, chat mode, reply length, persona, reset, delete |

### Long-press a turn

| Action | Effect |
|--------|--------|
| Edit response | REST edit → re-curates memories |
| Rewind to here | Rolls back story from this turn |
| Copy | Clipboard |
| Replay | WebSocket replay → new variants |

### Time skip sheet

**File:** `widgets/advance_time_button.dart`

| Option | What it sends |
|--------|---------------|
| Quiet moment | `continue` (no advance) |
| Hours / Day / Days / Season | `continue` + advance key |

---

## Lore Tome (Chronicle)

**File:** `lib/features/chronicle/presentation/chronicle_screen.dart`  
**Route:** `/chronicle/:instanceId`

Opens on **Recap** tab by default.

### The seven tabs (horizontal scroll)

```text
┌────────┬──────────┬─────────┬─────────┬────────┬───────┬─────────┐
│ Recap  │ Timeline │ Echoes  │ Almanac │ Places │ Bonds │ Threads │
└────────┴──────────┴─────────┴─────────┴─────────┴───────┴─────────┘
   ▲
   └── landing tab ("Story so far")
```

| Tab | What you see | API |
|-----|--------------|-----|
| **Recap** | Where/when pills, story spine, open threads, bond snapshot | `GET /chronicle/recap/:id` |
| **Timeline** | Full turn history (paginated) | `GET /chronicle/events/:id` |
| **Echoes** | Searchable memories + filters | `GET /chronicle/memories/:id` |
| **Almanac** | Story calendar, dated events, timeline switcher | `GET /chronicle/calendar/:id` |
| **Places** | Visited locations, "you are here" | `GET /chronicle/locations/:id` |
| **Bonds** | Relationship meters + moments | `GET /chronicle/relationships/:id` |
| **Threads** | Open & resolved promises | `GET /chronicle/threads/:id` |

Data loads **lazily** — first visit to a tab fetches that data.

### Echoes tab (memory search)

**File:** `widgets/echoes_filter_bar.dart`

- **Search bar** — full-text over memory text (submit to search)
- **Chips:** Unresolved, Important, types (relationship, promise, secret, …)
- **Memory cards** — edit or delete

### Almanac tab

**File:** `widgets/almanac_view.dart`

- **Present moment** — current in-world date
- **Realities** — switch timeline branch (confirm dialog)
- **Dated sections** — events grouped by story date
- **Travel markers** — "Traveled from X to Y"
- **Milestone / time jump markers**

### Places tab → Location journal

**File:** `widgets/places_view.dart` → `location_journal_screen.dart`

Tap a place:
- Permanent facts about the location
- Current state (mutable)
- Memories tied to this place
- Timeline of moments there

### Bonds tab → Character memory + Side chat

**File:** `widgets/bonds_view.dart`

Tap a character card:
- **Character memory screen** — "what they remember about you"
- **Private chat icon** → side chat thread (separate from main story)

Side chat:
- Loads history via REST
- Sends via WebSocket `side_chat`
- Streams `side_chat_delta` / `side_chat_complete`

### Threads tab

**File:** `widgets/threads_view.dart`

- **Open** — unresolved promises/conflicts (same data fed to AI as "open threads")
- **Resolved** — recently closed threads

Read-only in UI.

---

## WebSocket messages (play)

**File:** `lib/core/network/ws_manager.dart`

### You send

| Action | Purpose |
|--------|---------|
| `load_instance` | Full resync after rewind/reset |
| `chat` | Your message |
| `continue` | AI continues / time skip |
| `replay` | Regenerate last AI reply |
| `side_chat` | Private character message |

### Server pushes

| Type | You notice |
|------|------------|
| `generation_delta` | Text streaming |
| `generation_complete` | Turn done |
| `memories_curated` | New memories (for tap-to-read) |
| `character_codex_updated` | Bond meters refresh |
| `milestone_unlocked` | Toast |
| `side_chat_delta/complete` | Private chat streaming |

---

## Gameplay behavior summary

| You want to… | Do this |
|--------------|---------|
| Advance story | Type or tap choice chip or Continue |
| Skip time | Empty composer → clock button |
| See relationship status | Bond rail or Bonds tab |
| Find an old memory | Echoes search or tap name in prose |
| See story so far | Recap tab |
| Visit a place's history | Places → tap place |
| Talk privately to NPC | Bonds → private chat icon |
| Fix a bad AI line | Long-press → edit or replay |
| Undo many turns | Long-press → rewind |
| Switch alternate timeline | Almanac → Realities |

---

## What the frontend does NOT do

- **Does not** build the AI prompt (all server-side)
- **Does not** store the full 10k turn history locally (bounded cache + Chronicle pagination)
- **Does not** run memory extraction (worker job)
- **In-play memory lens** is a quick preview — Lore Tome has the authoritative full set
