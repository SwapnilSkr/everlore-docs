# Client Code Reference

`everlore/lib/` — Flutter app files grouped by folder (~98 Dart files).

---

## Hot path (read these first)

```text
PlayScreen → PlayCubit → WsManager → server
ChronicleScreen → ChronicleCubit → ChronicleRepository → server
SideChatScreen → WsManager (side_chat actions)
```

---

## `lib/features/play/` — live game

| File | Purpose | Optimizations |
|------|---------|---------------|
| `state/play_cubit.dart` | WS subscriptions, streaming, replay, milestones, rewind | **12 stream subs**; trim **100 events / 50 memories**; generation/replay **watchdogs**; stream reveal timers; `LocalDb` optimistic cache |
| `presentation/play_screen.dart` | Main UI shell, realm menu, settings/thoughts sheets | `BlocProvider`; auto-scroll on new events |
| `widgets/narrative_bubble.dart` | Turn rendering, entity links, replay UI | Stateless + `_ProseText` gesture recognizers; reduced-motion aware |
| `widgets/player_input.dart` | Composer + send | Disables while generating |
| `widgets/bond_rail.dart` | Relationship rings for cast | `CustomPainter`; max 5; dim absent characters |
| `widgets/choice_chips.dart` | Tap-to-play suggestions | Prefills composer only (no auto-send) |
| `widgets/advance_time_button.dart` | Time skip sheet | `continue` + advance payload |
| `widgets/world_state_bar.dart` | Expandable stat HUD | Animated deltas from cubit |
| `widgets/milestone_toast.dart` | Milestone celebration | Keyed animation by stamp |
| `widgets/story_timeline_sheet.dart` | Milestone list modal | Links to Chronicle |
| `widgets/bond_meters.dart` | Full meter detail in bond sheet | — |

---

## `lib/features/chronicle/` — Lore Tome

| File | Purpose | Optimizations |
|------|---------|---------------|
| `state/chronicle_cubit.dart` | 7-tab state, lazy loads, Echoes filters | **Lazy tab fetch**; optimistic memory edit map |
| `data/chronicle_repository.dart` | All `/chronicle/*` REST calls | Static methods; no cache (fresh per tab) |
| `presentation/chronicle_screen.dart` | Tab bar + content switcher | Default **Recap** tab |
| `presentation/widgets/recap_view.dart` | Story-so-far landing | Read-only |
| `presentation/widgets/almanac_view.dart` | Calendar + timeline switcher | PUT active timeline on switch |
| `presentation/widgets/places_view.dart` | Places index | Nav to journal |
| `presentation/widgets/bonds_view.dart` | Relationship ledger | Nav to memory + side chat |
| `presentation/widgets/threads_view.dart` | Open/resolved promises | Read-only |
| `presentation/widgets/echoes_filter_bar.dart` | Search + filter chips | Submit-triggered reload |
| `presentation/widgets/memory_card.dart` | Echoes row | Edit/delete actions |
| `presentation/widgets/edit_dialog.dart` | Memory editor form | Returns map to cubit |
| `presentation/location_journal_screen.dart` | Per-place history | On-demand fetch |
| `presentation/character_memory_screen.dart` | What character remembers | On-demand fetch |
| `presentation/side_chat_screen.dart` | Private NPC thread | Direct **WsManager** subs; local turn list |

### Chronicle DTOs (`data/`)

| File | API backing |
|------|-------------|
| `recap_data.dart` | `/chronicle/recap` |
| `calendar_data.dart` | `/chronicle/calendar` |
| `location_journal.dart` | `/chronicle/locations/...` |
| `relationship_ledger.dart` | `/chronicle/relationships` |
| `threads_data.dart` | `/chronicle/threads` |
| `side_chat_data.dart` | `/chronicle/side-chats` |

---

## `lib/core/network/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `ws_manager.dart` | **Singleton** play WebSocket | **13 broadcast streams**; connection epoch anti-stale; exponential reconnect (max 10); **offline message queue**; ping keepalive |
| `api_client.dart` | JWT HTTP wrapper | Shared `http.Client` |

### WebSocket actions (outbound)

`load_instance`, `chat`, `continue`, `replay`, `side_chat`, `ping`

### WebSocket types (inbound)

`connected`, `instance_loaded`, `generation_delta`, `generation_complete`, `generation_stream_end`, `memories_curated`, `character_codex_updated`, `milestone_unlocked`, `replay_delta`, `replay_complete`, `side_chat_delta`, `side_chat_complete`, `side_chat_error`, `generation_failed`, `error`

---

## `lib/core/auth/` & `storage/`

| File | Purpose | Optimizations |
|------|---------|---------------|
| `auth_service.dart` | Login/logout, session epoch notifier | Triggers feed refresh on auth change |
| `google_auth_service.dart` | Google Sign-In platform wrapper | — |
| `secure_storage.dart` | JWT + user JSON | Encrypted |
| `local_db.dart` | SQLite event cache | **200 event limit** per instance |
| `config/env.dart` | API/WS base URLs | Android localhost remap in debug |

---

## `lib/shared/models/`

| File | Purpose |
|------|---------|
| `event.dart` | Turn: prose, choices, replay variants, travel |
| `memory.dart` | Memory atom; `concerns(name)` for in-play lens |
| `world_instance.dart` | Instance snapshot + milestones |
| `character_profile.dart` | Codex + relationship meters |
| `world_template.dart` | Template blueprint |
| `user.dart` | Auth user + preferences |
| `persona.dart` | Player persona overlay |
| `realm_play_status.dart` | Multi-playthrough picker DTO |

Details: [../client/shared/models.md](../client/shared/models.md)

---

## `lib/features/home/` — Realms tab

| File | Purpose |
|------|---------|
| `state/home_cubit.dart` | In-progress instances list |
| `data/home_repository.dart` | Instance API + `realmChanges` broadcast |
| `presentation/home_screen.dart` | Grouped realm cards |
| `presentation/realm_playthroughs_screen.dart` | Pick/create story for one world |
| `presentation/realm_entry_flow.dart` | Continue vs new playthrough sheet |

---

## `lib/features/creator/` — Forge

| File | Purpose |
|------|---------|
| `state/forge_world_cubit.dart` | World wizard state |
| `state/create_character_cubit.dart` | Character wizard |
| `presentation/forge_world_screen.dart` | World creation UI |
| `presentation/create_character_screen.dart` | Character creation |
| `data/creator_repository.dart` | Template CRUD + image gen API |

---

## `lib/features/templates/` — Discover

| File | Purpose |
|------|---------|
| `presentation/browse_screen.dart` | Template grid |
| `presentation/template_detail_screen.dart` | Detail + start play |
| `data/template_repository.dart` | Published templates API |
| `data/interest_ranking.dart` | Client-side interest boost (signed-out) |

---

## `lib/features/personas/`

| File | Purpose |
|------|---------|
| `data/persona_repository.dart` | Persona CRUD + in-memory cache |
| `presentation/personas_screen.dart` | Persona management UI |

---

## `lib/app/` & `screens/`

| File | Purpose |
|------|---------|
| `app/theme/nexus_theme.dart` | **EverloreTheme** design tokens |
| `app_routes.dart` | GoRouter table + shell branches |
| `main.dart` | App entry |
| `screens/splash_screen.dart` | Animated splash + auth gate |
| `screens/auth_screen.dart` | Email / phone / Google auth |
| `screens/onboarding_screen.dart` | Name, gender, interests |
| `screens/discover_screen.dart` | Explore tab feed |

---

## `lib/shared/widgets/` (highlights)

| File | Purpose |
|------|---------|
| `everlore_nav_bar.dart` | Shell scaffold + bottom nav |
| `everlore_session_loader.dart` | Blocking forged loader |
| `neu.dart` | Neumorphic auth components |
| `realm_backdrop.dart` | Blurred cover background in play |

---

## Client optimizations summary

| Technique | Where | Why |
|-----------|-------|-----|
| WS singleton | `WsManager` | One connection for play + side chat |
| Event/memory trim | `PlayCubit` | Bound RAM on long sessions |
| SQLite cache | `LocalDb` | Instant resume offline/slow network |
| Lazy Chronicle tabs | `ChronicleCubit` | Don't fetch 7 APIs on open |
| In-play memory lens | `PlayScreen` | No API — filter local 50 memories |
| Stream reveal | `PlayCubit` | Smoother perceived streaming |
| Watchdog timers | `PlayCubit` | Recover stuck generation |
| `sessionEpoch` | `AuthService` | Refresh discover/realms after login |

---

## What the client deliberately does NOT do

- Build AI prompts or run RAG
- Store full 10k turn history locally
- Enforce side-chat privacy (server does — client uses correct endpoints)
- Run memory extraction (waits for `memories_curated` push)
