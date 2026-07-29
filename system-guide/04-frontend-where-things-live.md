# Frontend — Where Things Live

Map of the Flutter app: screens, buttons, and when they appear.

---

## App structure (high level)

```text
┌──────────────────────────────────────────────────────────┐
│  Shell nav: Explore · Realms · [+] · Worlds · Personas   │
└──────────────────────────────────────────────────────────┘
         │
         ├── Explore (/discover) → browse worlds
         ├── Realms (/) → your playthroughs
         ├── Worlds (/my-worlds) → forge / manage templates
         ├── Personas (/personas) → player personas
         ├── Create (+) → Forge a World | Create a Character
         └── Over shell:
               Play → /play/:instanceId
               Realm overview → /realm/:instanceId
               Chronicle → /chronicle/:instanceId
               Membership → /membership
```

---

## Play screen (the game)

**File:** `lib/features/play/presentation/play_screen.dart`

### What you always see (when loaded)

| UI | Location | Notes |
|----|----------|-------|
| Back | Top left | Exit play |
| Realm / menu ⋮ | Top | Realm overview + Chronicle, timeline, thoughts, settings |
| Connection dot | Header | Green = WebSocket live |
| Chat bubbles | Center scroll | Your messages + AI narration |
| Text composer | Bottom | Type or use prefilled chips |

### What appears conditionally

| UI | When it shows | When it hides |
|----|---------------|---------------|
| **World stat bar** | World has RPG stats | Stat-less character worlds |
| **Bond rail** | ≥1 side character with relationship meters | No bonded NPCs yet |
| **Choice chips** | Latest turn done, server sent choices for active variant | Generating, replaying, disconnected |
| **World Actions button** | Composer empty, connected | You typed text, generating |
| **"Older history" link** | More than 20 events exist | All history loaded |
| **Milestone toast** | Server unlocked a milestone | Auto-dismisses |
| **Protagonist onboarding** | GM world, no player character yet | After you name yourself |
| **Error bar** | Something failed | You dismiss |

### World Actions (replaces Advance-Time-only affordance)

**File:** `widgets/world_actions_button.dart` (wired from `player_input.dart`)

| Action | What happens |
|--------|----------------|
| Continue story | WS `continue` |
| Later today / Tomorrow / A few days / Next season | WS `continue` + advance |
| Travel to… | WS `world_action` travel (destination, companions, optional time) |
| Set a relationship | REST kinship write |
| Review story details | Open relation candidates (accept / reject / defer) |

Known destinations come from Chronicle locations. Kinship / candidates: see [chronicle-feature.md](../client/features/chronicle-feature.md).

### Bond rail (relationship presence)

**File:** `widgets/bond_rail.dart`

- Up to **5** characters shown as circular meters
- Color = dominant bond (trust green, affection rose, fear violet, rivalry amber)
- **Dimmed** if character not in current scene — matches **canonical name + aliases** against `presentCharacters` (null list = everyone present)
- **Tap** → action sheet with **Here now / Elsewhere** tag; **Approach** when present, **Seek out** when elsewhere

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
| Edit response | REST edit → re-curates memories; **regenerates chips** when AI prose changed |
| Rewind to here | Rolls back story from this turn |
| Copy | Clipboard |
| Replay | WebSocket replay → new variant with its own chips + presence |
| Replay arrows | Browse variants locally; chips/presence swap with prose; commit before next send |

---

## Lore Tome (Chronicle)

**File:** `lib/features/chronicle/presentation/chronicle_screen.dart`  
**Route:** `/chronicle/:instanceId`

Opens on **Recap** tab by default.

### The seven tabs (horizontal scroll)

```text
┌────────┬──────────┬─────────┬─────────┬────────┬───────┬─────────┐
│ Recap  │ Timeline │ Echoes  │ Almanac │ Places │ Bonds │ Threads │
└────────┴──────────┴─────────┴─────────┴────────┴───────┴─────────┘
   ▲
   └── landing tab ("Story so far")
```

| Tab | What you see | API |
|-----|--------------|-----|
| **Recap** | Where/when pills, story spine, open threads, bond snapshot | `GET /chronicle/recap/:id` |
| **Timeline** | Full turn history (paginated) | `GET /chronicle/events/:id` |
| **Echoes** | Searchable memories + filters | `GET /chronicle/memories/:id` |
| **Almanac** | Story calendar, dated events, timeline switcher | `GET /chronicle/calendar/:id` |
| **Places** | Nested atlas / fog-of-war tree, "you are here" | `GET /chronicle/locations/:id` |
| **Bonds** | Relationship meters + moments | `GET /chronicle/relationships/:id` |
| **Threads** | Open & resolved promises | `GET /chronicle/threads/:id` |

Data loads **lazily** — first visit to a tab fetches that data.

### Bonds tab → Character memory + Side chat

**File:** `widgets/bonds_view.dart`

Tap a character card:
- **Character memory screen** — "what they remember about you"
- **Private chat icon** → side chat thread (separate from main story)

Side chat loads history via REST; sends via WebSocket `side_chat`.

---

## Membership & Ink

**Route:** `/membership` · **File:** `lib/features/billing/presentation/billing_screen.dart`

Shows ledger-backed Ink balance, Premium / Creator plans, Ink packs, and (when enabled) Google Play or test checkout. Full detail: [billing-feature.md](../client/features/billing-feature.md).

---

## WebSocket messages (play)

**File:** `lib/core/network/ws_manager.dart`

### You send

| Action | Purpose |
|--------|---------|
| `load_instance` | Full resync after rewind/reset |
| `chat` | Your message |
| `continue` | AI continues / time skip |
| `world_action` | Structured travel (and related server kinds) |
| `replay` | Regenerate last AI reply |
| `side_chat` | Private character message |

### Server pushes

| Type | You notice |
|------|------------|
| `generation_delta` | **Prose** streaming (not a JSON turn body) |
| `generation_complete` / `generation_stream_end` | Turn done |
| `replay_delta` / `replay_complete` | Replay streaming; chips + presence per variant |
| `memories_curated` | New memories (for tap-to-read) |
| `character_codex_updated` | Bond meters refresh |
| `milestone_unlocked` | Toast |
| `side_chat_delta/complete` | Private chat streaming |

---

## Gameplay behavior summary

| You want to… | Do this |
|--------------|---------|
| Advance story | Type or tap choice chip or Continue |
| Skip time / travel / set kinship | Empty composer → World Actions |
| Review narrator proposals | World Actions → Review story details |
| See relationship status | Bond rail or Bonds tab |
| Find an old memory | Echoes search or tap name in prose |
| See story so far | Recap tab |
| Visit a place's history | Places → tap place |
| Talk privately to NPC | Bonds → private chat icon |
| Fix a bad AI line | Long-press → edit or replay |
| Undo many turns | Long-press → rewind |
| Buy Ink / change plan | Membership (`/membership`) |
| Switch alternate timeline | Almanac → Realities |

---

## What the frontend does NOT do

- **Does not** build the AI prompt (all server-side)
- **Does not** store the full 10k turn history locally (bounded cache + Chronicle pagination)
- **Does not** run memory extraction (worker job)
- **Does not** settle Ink (server reserve/settle)
- **In-play memory lens** is a quick preview — Lore Tome has the authoritative full set
