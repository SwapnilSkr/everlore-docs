# Visual Maps

Quick diagrams for where things live in the app and how data flows.

---

## Play screen layout

```text
┌─────────────────────────────────────────────────────────────┐
│  ← Back          [connection ●]                    ⋮ Menu   │
├─────────────────────────────────────────────────────────────┤
│  ▼ World stats (if gamified)                                │
├─────────────────────────────────────────────────────────────┤
│  ○ ○ ○ ○ ○  Bond rail (tap → approach / ask / memories)     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────┐                       │
│   │  AI narration bubble            │                       │
│   │  Names = gold underline         │                       │
│   │  Places = dotted underline      │                       │
│   │  [Continue] [Replay]            │  ← latest turn only   │
│   └─────────────────────────────────┘                       │
│                                                             │
│   ┌─────────────────────────────────┐                       │
│   │  Your message                   │                       │
│   └─────────────────────────────────┘                       │
│                                                             │
│   [ View older history in Chronicle ]  ← if >20 turns       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  [ chip ] [ chip ] [ chip ]   Choice chips (optional)       │
├─────────────────────────────────────────────────────────────┤
│  [🕐]  Type your message...                          [Send] │
│   ▲                                                         │
│   └── Time skip (only when box empty)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Realm menu (⋮)

```text
┌──────────────────────────┐
│  Chronicle          →    │  Lore Tome (7 tabs)
│  Story Timeline     →    │  Milestones sheet
│  Thoughts           →    │  Cast + focus + edit cards
│  Settings           →    │  POV, mode, length, reset
└──────────────────────────┘
```

---

## Lore Tome (Chronicle)

```text
┌─────────────────────────────────────────────────────────────┐
│  ← Back to Play                                             │
├─────────────────────────────────────────────────────────────┤
│ Recap │ Timeline │ Echoes │ Almanac │ Places │ Bonds │ Threads │
│  ▲                                                          │
│  └── default tab                                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   (tab content — see below)                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Tab contents at a glance

```text
RECAP          TIMELINE         ECHOES
┌──────────┐   ┌──────────┐    ┌──────────┐
│ Where/   │   │ Turn 1   │    │ [search] │
│ When     │   │ Turn 2   │    │ filters  │
│ Story    │   │ Turn 3   │    │ memory   │
│ spine    │   │ ...      │    │ cards    │
│ Open     │   │ travel   │    │ edit/del │
│ threads  │   │ markers  │    └──────────┘
│ Bonds    │   └──────────┘
└──────────┘

ALMANAC        PLACES           BONDS
┌──────────┐   ┌──────────┐    ┌──────────┐
│ Present  │   │ You are  │    │ Char A   │──→ char memories
│ moment   │   │ here     │    │ meters   │──→ private chat 💬
│ Realities│   │ Place B  │──→ │ Char B   │
│ switcher │   │ Place C  │    │ ...      │
│ Dated    │   └──────────┘    └──────────┘
│ events   │        │
└──────────┘        └──→ Location journal
                              (facts, echoes, moments)

THREADS
┌──────────┐
│ Open     │
│ promises │
│ Resolved │
└──────────┘
```

---

## Side chat (from Bonds only)

```text
Bonds tab → tap 💬 on character
        │
        ▼
┌─────────────────────────────┐
│  ← Private chat with Mira   │
├─────────────────────────────┤
│  (in-character thread)      │
│  NOT in main Play timeline  │
├─────────────────────────────┤
│  Type message...      [Send]│
└─────────────────────────────┘
```

---

## Backend: one turn data flow

```text
     YOU
      │
      ▼
┌───────────┐    enqueue     ┌─────────────────┐
│ WebSocket │ ──────────────►│ Generation job  │
└───────────┘                └────────┬────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
             Context packet      Prompt           Stream AI
             (search first)     (bounded)         reply
                    │                 │                 │
                    └────────┬────────┴────────┬────────┘
                             ▼                 ▼
                        Save EVENT      Update codex
                        + time/place    + entity graph
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        Memory job    Summary job?    Push to app
        (async)       (every 12)      (stream + memories)
```

---

## Memory layers (what the AI reads)

```text
         ┌─────────────────────────────────────┐
         │         STATIC (cached)               │
         │  World lore, voice, format rules      │
         └─────────────────────────────────────┘
         ┌─────────────────────────────────────┐
         │         DYNAMIC (per turn)            │
         │  Codex cards ────┐                    │
         │  Memories ───────┼── token budgets    │
         │  Lore hits ──────┤                    │
         │  Open threads ───┤                    │
         │  Past chapters ──┘                    │
         │  Time + place + timeline              │
         └─────────────────────────────────────┘
         ┌─────────────────────────────────────┐
         │  Scene summary (1)                    │
         │  Recent turns (6, min 1000 tokens)    │
         │  YOUR MESSAGE                           │
         └─────────────────────────────────────┘
```

---

## When UI elements appear (decision tree)

```text
Bond rail?
  └─ Any side character with relationship meters? → YES show (max 5)

Choice chips?
  └─ Latest turn settled?
      └─ Server sent choices?
          └─ Not generating/replaying? → YES show

Time skip button?
  └─ Composer empty?
      └─ Connected? → YES show

Tappable names in prose?
  └─ AI text finished streaming? → YES (not while generating)

Private chat entry?
  └─ Only from Bonds tab → NOT from Play bond sheet

Recap vs Timeline content?
  └─ Recap = compressed summary + threads + bonds snapshot
  └─ Timeline = raw turn-by-turn (includes travel headers)
```

---

## Privacy boundary (main vs side chat)

```text
                    MAIN STORY                    SIDE CHAT
                    ──────────                    ─────────
Events in           Play, Timeline,               Side chat
                    Recap, Calendar               screen only

Memories in         Echoes, RAG prompt,           Scoped to
                    Recap, main narration         character + player
                                                  (knowers list)

Advances time?      YES                           NO

Updates bonds?      YES                           YES (that character)

In scene summaries? YES                           NO
```
