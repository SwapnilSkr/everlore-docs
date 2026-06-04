# Everlore Docs

Central documentation hub for the Everlore Flutter client and Elysia server.

## Structure

### Client Docs

Located in [`client/`](./client)

- [Architecture Overview](./client/architecture/overview.md)
- [Routing](./client/architecture/routing.md)
- [State Management](./client/architecture/state-management.md)
- [Core Layer](./client/core/core-layer.md)
- [Getting Started](./client/guides/getting-started.md)
- [Shared Models](./client/shared/models.md)

Auth-related client details:
- Google Sign-In and phone OTP entry flow: [Architecture Overview](./client/architecture/overview.md)
- Session persistence, WebSocket auth, secure storage: [Core Layer](./client/core/core-layer.md)
- Local environment setup: [Getting Started](./client/guides/getting-started.md)

### Server Docs

Located in [`server/`](./server)

- [Server README](./server/README.md)
- [Architecture Overview](./server/OVERVIEW.md)
- [API](./server/API.md)
- [Configuration](./server/CONFIGURATION.md)
- [Data Model](./server/DATA_MODEL.md)
- [Services](./server/SERVICES.md)
- [Workers](./server/WORKERS.md)
- [Memory & Codex Architecture](./server/MEMORY_ARCHITECTURE.md)
- [Infrastructure](./server/INFRASTRUCTURE.md)
- [Security](./server/SECURITY.md)

### Infinite Memory Docs

Located in [`memory/`](./memory)

- [Memory Docs README](./memory/README.md)
- [Infinite Memory Architecture](./memory/MEMORY_ARCHITECTURE.md)
- [Projection And Mutation Model](./memory/PROJECTION_AND_MUTATION_MODEL.md)
- [Calendar, Timelines, And Locations](./memory/CALENDAR_TIMELINES_AND_LOCATIONS.md)
- [Features Enabled By Infinite Memory](./memory/FEATURES_ON_TOP.md)
- [Infinite Memory Checklist](./memory/CHECKLIST.md)

Auth-related server details:
- HTTP and WebSocket auth contract: [API](./server/API.md)
- Google and Twilio env vars: [Configuration](./server/CONFIGURATION.md)
- JWT, Google verification, Twilio Verify, and rate limits: [Security](./server/SECURITY.md)
- User fields such as `phone`, `google_sub`, and provider metadata: [Data Model](./server/DATA_MODEL.md)

## Current Auth Summary

Everlore now uses a unified backend JWT session model for:

- Email/password login and registration
- Google Sign-In token exchange
- Twilio Verify phone OTP

The Flutter app stores the returned JWT in secure storage and reuses it for both REST and WebSocket authentication.
