# Everlore Docs

Central documentation hub for the Everlore Flutter client and Elysia server.

## Structure

### Client Docs

Located in [`client/`](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client)

- [Architecture Overview](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/architecture/overview.md)
- [Routing](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/architecture/routing.md)
- [State Management](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/architecture/state-management.md)
- [Core Layer](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/core/core-layer.md)
- [Getting Started](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/guides/getting-started.md)
- [Shared Models](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/shared/models.md)

Auth-related client details:
- Google Sign-In and phone OTP entry flow: [Architecture Overview](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/architecture/overview.md)
- Session persistence, WebSocket auth, secure storage: [Core Layer](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/core/core-layer.md)
- Local environment setup: [Getting Started](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/client/guides/getting-started.md)

### Server Docs

Located in [`server/`](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server)

- [Server README](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/README.md)
- [Architecture Overview](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/OVERVIEW.md)
- [API](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/API.md)
- [Configuration](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/CONFIGURATION.md)
- [Data Model](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/DATA_MODEL.md)
- [Services](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/SERVICES.md)
- [Workers](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/WORKERS.md)
- [Infrastructure](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/INFRASTRUCTURE.md)
- [Security](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/SECURITY.md)

Auth-related server details:
- HTTP and WebSocket auth contract: [API](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/API.md)
- Google and Twilio env vars: [Configuration](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/CONFIGURATION.md)
- JWT, Google verification, Twilio Verify, and rate limits: [Security](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/SECURITY.md)
- User fields such as `phone`, `google_sub`, and provider metadata: [Data Model](/Users/fatman/Desktop/codes/rpg-ai/everlore-docs/server/DATA_MODEL.md)

## Current Auth Summary

Everlore now uses a unified backend JWT session model for:

- Email/password login and registration
- Google Sign-In token exchange
- Twilio Verify phone OTP

The Flutter app stores the returned JWT in secure storage and reuses it for both REST and WebSocket authentication.
