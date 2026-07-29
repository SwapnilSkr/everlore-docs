# Everlore Client Documentation

Flutter app docs for Play, Lore Tome, Membership & Ink, Forge, and shell navigation.

**Product guide:** [../system-guide/04-frontend-where-things-live.md](../system-guide/04-frontend-where-things-live.md)  
**File-by-file code reference:** [../code-reference/CLIENT.md](../code-reference/CLIENT.md)  
**Visual pixel guide:** [../visual-guide/](../visual-guide/) (may lag behind — prefer system-guide for behavior)

---

## Architecture

| Document | Description |
|----------|-------------|
| [architecture/overview.md](./architecture/overview.md) | Stack, folders, data flow |
| [architecture/routing.md](./architecture/routing.md) | All routes (4 shell tabs + overlays) |
| [architecture/state-management.md](./architecture/state-management.md) | Cubit patterns |
| [architecture/theme.md](./architecture/theme.md) | EverloreTheme tokens |
| [core/core-layer.md](./core/core-layer.md) | Auth, API, WebSocket, SQLite |

## Features

| Document | Description |
|----------|-------------|
| [features/play-feature.md](./features/play-feature.md) | Game screen, World Actions, bond rail, WS |
| [features/chronicle-feature.md](./features/chronicle-feature.md) | Lore Tome — 7 tabs + kinship / candidates |
| [features/billing-feature.md](./features/billing-feature.md) | Membership & Ink (`/billing/*`) |
| [features/home-feature.md](./features/home-feature.md) | Realms tab |
| [features/templates-feature.md](./features/templates-feature.md) | Discover + template detail |

## Shared

| Document | Description |
|----------|-------------|
| [shared/models.md](./shared/models.md) | Event, Memory, Instance, … |
| [shared/widgets.md](./shared/widgets.md) | Reusable widgets |

## Guides

| Document | Description |
|----------|-------------|
| [guides/getting-started.md](./guides/getting-started.md) | Local setup |

---

## Quick map

```text
Shell: Explore · Realms · Worlds · Personas  (+ Create)
Play (/play/:id)           → real-time story, World Actions, WsManager
Realm (/realm/:id)         → calm playthrough overview
Chronicle (/chronicle/:id) → Lore Tome, REST lazy tabs
Membership (/membership)   → Ink wallet + Play purchases
Side chat                  → Bonds tab → SideChatScreen (WS)
Forge                      → /my-worlds/forge, /characters/new
```
