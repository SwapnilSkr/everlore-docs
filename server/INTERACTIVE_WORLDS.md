# Interactive Worlds: Backend, Database, and Assets

## Ownership boundary

The API owns canonical interactive-world state. Flutter renders the returned world graph and can optimistically animate a move, but does not decide whether a route is unlocked or whether a quest action succeeded.

```text
interactive_worlds        authored, revisioned world definition
interactive_world_states  one durable state record per player world instance
world_events / memories   existing canonical consequence and retrieval layers
S3 + CloudFront           immutable visual assets referenced by manifest keys
```

## Collections

### `interactive_worlds`

One document per authored world key. Ashen Circuit is `ashen-circuit`.

- `version`: increments only when an authored definition changes compatibly.
- `map_style`: runtime map renderer, world-specific skin, terrain asset id,
  and a landmark scene graph. Map geometry, routes, locks, hit polygons, and
  location state never come from a baked location image.
- `assets`: logical id, immutable S3 key, asset role, and revision.
- `locations`: map coordinates, routes, scene asset reference, and optional unlock flag.

The first endpoint request idempotently seeds the Ashen Circuit definition. In production this becomes an authoring/publish workflow; it must not mutate a published revision in place once players are using it.

### `interactive_world_states`

One record per `instance_id + world_key`, with a unique compound index.

- current location;
- unlocked locations;
- quest flags;
- seen scenes;
- monotonic sequence; and
- timestamps.

This state is deliberately compact. Every resolved interactive action now also
appends an `interactive_world` record to the existing `world_events` ledger,
with a deterministic summary and flag mutation where applicable. NPC/context
retrieval can therefore use the same canonical event history as chat. The first
quest promotions are live too: the first Arena witness, Cassian trust, Nara
meeting, and Barracks seal study each create one importance-7 memory atom. They are marked as
unresolved threads, and later repeats do not duplicate them.

## Initial API

All instance endpoints require the authenticated owner of that world instance.
Flutter opens the persistent renderer with
`/interactive/ashen-circuit/lab?instanceId=<id>`; omitting the parameter is
only the local authoring preview.

| Endpoint | Purpose |
|---|---|
| `GET /interactive-worlds/:worldKey` | Definition plus CDN-resolved assets |
| `GET /interactive-worlds/:worldKey/instances/:instanceId` | Definition and initialise/read player state |
| `POST /interactive-worlds/:worldKey/instances/:instanceId/actions` | Resolve a typed action |

Initial actions are `move` and `choose`. `move` validates both the authored route and its unlock rule. `choose:witness_arena` records the arena scene and unlocks Ember Market; `choose:trust_cassian` opens Fighters’ Barracks after the summons; `choose:meet_nara` records the Market agent encounter; `choose:inspect_barracks` resolves the ember-letter and opens Ash Gate; and `choose:challenge_ash_gate` opens the Cinder Underway. More choices must be added as named, validated actions; the client must not submit arbitrary flag mutations.

## S3 / CloudFront asset rule

All production interactive assets live under this durable prefix:

```text
interactive-worlds/ashen-circuit/
  maps/cinder-ward/terrain-v1.png
  maps/cinder-ward/landmarks/cinder-arena/base-v1.png
  maps/cinder-ward/landmarks/cinder-arena/embers-v1.png
  maps/cinder-ward/landmarks/ash-gate/base-v1.png
  locations/cinder-arena/dusk-background-v1.png
  characters/cassian-vale/neutral-v1.png
  locations/ember-market/night-background-v1.png
  characters/nara-voss/neutral-v1.png
  locations/fighters-barracks/night-background-v1.png
  locations/ash-gate/night-background-v1.png
```

The S3 bucket remains private. `storageService.urlForKey()` resolves manifest keys only through `CDN_BASE_URL`; clients never see an S3 origin URL and never upload directly to the bucket. New art creates a new versioned object key; never overwrite a published visual asset. A `landmark_scene_graph` map style references a terrain asset plus explicit landmark layers and hit polygons. After an asset is uploaded through the server’s existing storage service, update the authored manifest revision.

When a prototype asset is explicitly retired before release, remove both its
manifest entry and exact S3 key with the server-side retirement script. Do not
reuse retired keys; replacement assets always receive new, versioned keys.

Required deployment configuration already exists in `.env.example`: `AWS_REGION`, `S3_BUCKET`, standard AWS credentials/role, and `CDN_BASE_URL`. CloudFront Origin Access Control and public-access blocking are required.

The repeatable uploader is `bun run upload:interactive-assets` in
`everlore-server`. It uploads the seven source assets from
`../everlore/assets/art/ashen_circuit` to their exact manifest keys. CI can set
`INTERACTIVE_ASSET_SOURCE_ROOT` when its checkout layout differs. The command
fails before writing if S3 or CloudFront configuration is absent. It probes
each revisioned key first and skips already-published objects, so rerunning it
cannot silently overwrite released artwork.

## Current implementation

Implemented in `everlore-server`:

- `src/models/interactive-world.model.ts`
- `src/services/interactive-world.service.ts`
- `src/routes/interactive-world.routes.ts`
- Mongo collection/index registrations
- CDN-only asset-key resolution in `storage.service.ts`

## Next integration steps

1. Add the Cinder Underway asset kit and its first resident as version 4.
2. Pass the selected play instance automatically from every intended entry point.
3. Add the same idempotent memory-promotion policy to Ash Gate branch choices,
   including explicit NPC witness lists.
4. Add an authenticated authoring endpoint that validates asset manifests and publishes a new immutable world version.
