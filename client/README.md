# Everlore Client Documentation

Flutter app docs for Play, Lore Tome, Forge, and shell navigation.

**Product guide:** [../system-guide/04-frontend-where-things-live.md](../system-guide/04-frontend-where-things-live.md)  
**File-by-file code reference:** [../code-reference/CLIENT.md](../code-reference/CLIENT.md)  
**Visual pixel guide:** [../visual-guide/](../visual-guide/) (may lag behind — prefer system-guide for behavior)

---

## Architecture

| Document | Description |
|----------|-------------|
| [architecture/overview.md](./architecture/overview.md) | Stack, folders, data flow |
| [architecture/routing.md](./architecture/routing.md) | All routes |
| [architecture/state-management.md](./architecture/state-management.md) | Cubit patterns |
| [architecture/theme.md](./architecture/theme.md) | EverloreTheme tokens |
| [core/core-layer.md](./core/core-layer.md) | Auth, API, WebSocket, SQLite |

## Features

| Document | Description |
|----------|-------------|
| [features/play-feature.md](./features/play-feature.md) | Game screen, bond rail, WS |
| [features/chronicle-feature.md](./features/chronicle-feature.md) | Lore Tome — 7 tabs |
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
Play (/play/:id)     → real-time story, WsManager
Chronicle (/chronicle/:id) → Lore Tome, REST lazy tabs
Side chat            → Bonds tab → SideChatScreen (WS)
Forge                → creator/ feature
Discover             → templates/ browse
```
