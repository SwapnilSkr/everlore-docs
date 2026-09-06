# Interactive Map Pipeline — painted terrain, modelled landmarks, living ambience

_Design decision + execution plan, locked in the September 2026 discussion that
followed the Iron Verdict art build. Supersedes nothing on disk — the current
renderer (`terrain_plate_scene_graph`) stays exactly as it is. What changes is
where landmark art comes from, where world definitions live, and what moves on
the map. Read `IRON_VERDICT_WORLD_BIBLE.md` §6 first for the socket rule and the
seating method; this doc is the wider architecture those findings imply._

## Status

| | |
| --- | --- |
| Decided | Painted terrain, **pre-rendered 3D** landmarks, ambience layers |
| Rejected for now | Real-time 3D terrain mesh (see *Rejected options*) |
| Blender | 5.2.1 LTS installed, headless verified — `bpy 5.2.1`, Python 3.13.13 |
| Scale target | 2 templates (medieval, isekai) × ~20 authored worlds each |
| Authoring | Owner-authored today; creator-authored later — that is what forces the data model change |

## Why this came up

Iron Verdict's map is 5 painted plates + 27 alpha-matted landmark sprites
composited on top. Building it exposed two complaints that sound like one problem
and are not:

1. **`everlore-server/src/worlds/iron-verdict.ts` is a hand-written TypeScript
   file.** That does not scale to 40 worlds and cannot survive creators.
2. **Compositing sprites onto a painting is fragile.** Seating one gate into a
   painted wall cost four regenerations and most of a day.

They have different answers. Treat them separately.

---

## 1. The finding that should drive everything

**Seven of the eight Corvane landmarks went into their sockets and needed only
scale tweaks. The eighth cost hours.**

The difference was not 2D vs 3D. It was that Marblegate had to **continue
geometry the plate had already painted** — a gate is a modification of a wall,
not an object standing near one. Every other landmark occupies open ground and
carries its own footprint.

> **RULE — a composited landmark may occupy empty space, but must never join a
> painted line.** If the fiction needs a gate in a wall or a bridge over a
> painted river, either the plate paints it (and the place becomes a `marker`
> hotspot), or the plate leaves open ground and the landmark stands in it.

Ravensgate Bridge is the next asset with this exact problem — painted river,
painted banks. Decide it by this rule *before* generating it.

The second finding, from the same fight: **the plate's camera is real but
undeclared.** It exists only implicitly inside the painting, so every sprite has
to be matched to it by eye or by measuring pixels. That is the root cause of the
camera mismatch, the in-plane rotation, and the hand-measured sockets. Fixing it
is the cheapest high-value change in this document.

---

## 2. Decision: painted terrain, modelled landmarks, pre-rendered

Terrain stays painted. The plates are carrying the visual quality and a naive
real-time 3D scene would look markedly worse than they do.

Landmarks move to **3D models rendered to PNG through the plate's declared
camera** — not rendered at runtime. One source, three outputs:

```
Blender  (parametric, script-driven, headless)
   ├── render at the plate camera        → map sprite PNG    ships today, no renderer change
   ├── render 36-frame turntable         → rotate-on-select  the premium beat
   └── export glTF                       → optional runtime 3D, later
```

What this buys, without touching the renderer:

- **Camera correct by construction.** The entire failure class from the gate
  disappears; it becomes a yaw value.
- **Real contact shadows and one lighting model.** The "floats slightly" flaw
  visible at high zoom is a lighting problem, not a placement problem, and it
  cannot be fixed from the placement side.
- **State variants become re-renders.** The 8 `_awakened` / `_sealed` /
  `_besieged` variants stop being regenerations that must match a silhouette by
  luck.
- **Rotation is free**, so a landmark can face the road it belongs to.

Topology quality barely matters here — these are viewed at ~156 px from a near-
fixed camera, never rigged. That removes the usual objection to AI-assisted 3D.

### Rotate-on-select without a 3D map

The premium interaction ("click a location and it revolves") does **not** require
a 3D map. In many games that rotation is a *detail view*, not the world. The
selection card already shows the landmark art (`_SelectionCard.artUrl`).

**v1 is a 36-frame turntable scrubbed on drag** — a small sprite sheet, zero 3D
runtime, works on every device, visually identical to a locked orbit. Free orbit
via glTF (`model_viewer_plus` or `flutter_scene`) is a later swap, and the models
already exist by then.

---

## 3. Asset economics

Per world, from `IRON_VERDICT_BATCH_PLAN.md` (96 assets):

| set | count | 3D? |
| --- | --- | --- |
| Landmark sprites | 27 | **yes** — the interactive set |
| State variants | 8 | **free** — re-render of the above |
| Terrain plates | 5 | no — stay painted |
| Scenes | 22 | never — illustrations |
| Portraits | 30 | never |
| Time-of-day scenes | 4 | never |

**58% of the asset budget stays 2D permanently.** Twenty-seven things get
modelled; eight more fall out for free.

### Reuse is what makes 20 worlds per template viable

- **12 archetypes** (village, farmstead, watchtower, ruined chapel, longhall,
  keep, mine head, quarry, wharf, manor, bridge, camp) already cover 30 locations
  in *one* world. Modelled once per template, they serve all 20.
- **~12 heroes per world** is the apparent 240-model problem. It is not, if
  heroes are **assemblies from a kit of parts**, never sculpts: wall runs, tower
  drums, conical/gabled roofs, arcades, arches, market stalls, cranes, plinths,
  banners. Build ~50 pieces; assemble heroes from them.
- Across 20 medieval worlds most heroes are archetypes with different dressing —
  an arena is an arena. The hero set collapses toward archetype + variation.

---

## 4. The Blender pipeline

**Generate geometry parametrically; do not rely on image-to-3D as the primary
path.** Style consistency across generated *meshes* is far harder to hold than
across generated *images*, where a prompt can be pinned. Forty worlds of drifting
texture work is a real cost.

Instead: **an agent writes Blender Python that constructs the mesh.** A gatehouse
is two cylinders, a wall segment, an arch boolean and a crenellation array —
deterministic, restylable, diffable, and it compounds with the kit-of-parts idea
because what is generated is an *assembly script*, not a hoped-for mesh.

Image-to-3D stays available for organic one-offs, curated by hand.

Verified working:

```bash
/Applications/Blender.app/Contents/MacOS/Blender -b --python-expr "..."   # headless, no GUI
```

Headless means the whole thing is scriptable and CI-able alongside
`scripts/process-interactive-assets.ts`.

### Storage

Same revisioned-key discipline as today — **a published key is never
overwritten**, new art is a new revision, because CloudFront caches by URL.

| artefact | where | why |
| --- | --- | --- |
| `.blend` source / assembly script | repo or cold S3 | the real source of truth |
| rendered sprite `.webp` | S3 → CloudFront | what ships now |
| turntable sheet | S3 → CloudFront | the select interaction |
| `.glb` | S3 | optional runtime 3D later |

---

## 5. The declared camera

Add the camera to `map_style` so it stops being implicit in the painting:

```ts
camera: { elevation: number, yaw: number, projection: 'ortho' | 'perspective' }
```

Everything downstream depends on this contract: Blender renders to it, the
placement engine projects world coordinates through it, and future plate prompts
are written against it. **Do this before modelling anything.**

---

## 6. Placement must stop being hand-measured

Hand-measuring sockets off a painting does not survive worlds we did not paint
one at a time. The seed of the fix already exists: `check-interactive-plate.ts`
autonomously found 11 sockets on the Corvane plate and accepted/rejected the
plate on housing coverage.

**Promote that audit into a placement engine.** With a declared camera and
automatic socket detection, a plate places its own landmarks and nobody measures
anything by hand. This — not meshes — is what stands between us and worlds that
are not constant.

---

## 7. World definitions must leave TypeScript

`InteractiveWorldDoc` is already a document, `map_style.renderer` is already
world data, and `ensureWorld` already upserts into Mongo. So `iron-verdict.ts` is
not the architecture — it is a **seed fixture that got treated as the source of
truth**. The gap is that there is no authoring path except editing TypeScript and
remembering to bump a constant.

⚠️ **Bigger blast radius than first assessed.** The file (289 lines, 33 location
rows) is imported by **five** call sites, and three of them are *tooling*:
`process-interactive-assets.ts`, `check-interactive-plate.ts` and
`iron-verdict-audit.ts` all read the world definition directly from TypeScript.
So it is a build-time dependency of the asset pipeline and both audits, not only
a runtime source. Moving worlds into documents means giving those three a
different input or they break. That is the actual work, and it is why this is
not a quick win.

⚠️ **The silent failure to preserve:** `ensureWorld` writes only when
`version < WORLD_VERSION`, deliberately, so published art never shifts under a
player mid-run. Edit a world and forget the bump and the server reloads, reports
no error, and serves the old data. This cost real time in September 2026 (code
said 18, database sat on 17). Any authoring tool must make the version bump
impossible to forget.

---

## 8. Ambience — the cheapest premium win in this document

The map is a still painting. Motion is likely most of the "premium" feeling, and
it costs almost nothing.

**The 3D read comes from which layer an effect lives in, not from the effect.**

Today everything is inside one transformed layer
(`InteractiveViewer → SizedBox(canvas) → Stack{plates, landmarks, fog}`).
Snow placed *there* scales with zoom — flakes become dinner plates at 4.5× and
read as stickers. Snow in **screen space above the viewer, drifting at a fraction
of the pan**, reads as weather between the camera and the world. Same particle
code; completely different perception. That parallax lag is the whole trick.

| effect | layer | needs | cost |
| --- | --- | --- | --- |
| Drifting cloud shadows | world | nothing | trivial |
| Snow / rain / ash | **screen** | nothing | trivial |
| Brazier + window flicker | world | point list per plate | small |
| Smoke plumes | world | point list | small |
| River flow | world | **water mask** | medium |

The first two need no new art and no pipeline work and do most of the lifting.

**Water's cost is the mask, not the animation.** Derive it in the asset pipeline
by colour segmentation + flood fill, emit a 1-bit mask beside the plate, and
eyeball it on the contact sheet that already gets generated. ⚠️ The threshold
must be **measured per plate**, never a constant — a global colour constant
breaks the first time a world has jade or blood-red water. Animation is then a
scrolling caustics overlay masked to that region, or an Impeller fragment shader
for the premium version. One scroll vector per plate is convincing at map scale;
a full flow field is not worth it.

⛔ **Do not add per-landmark parallax.** Offsetting sprites against the pan to
fake height would unseat every landmark we carefully placed, every frame. Get
depth from atmosphere layers only.

Ambience is **world data**, not a hardcoded renderer behaviour:

```ts
ambience: {
  weather: 'snow' | 'rain' | 'ash' | 'petals' | 'none',
  cloud_shadows: boolean,
  water: { flow: [number, number], strength: number },
}
```

Then drive it from the existing genre-aware calendar — snow because it is winter
*in the story*, ash because the mountain is erupting in chapter three. The map
starts tracking the narrative instead of decorating it.

**Performance:** one shared ticker (not per-effect controllers), `RepaintBoundary`
per layer, capped particle counts, and pause when the app backgrounds or the
selection sheet covers the map. ⚠️ Do not trust the emulator — it was already
logging `Skipped 171 frames` on a static plate with no animation at all. Verify
on a real device.

---

## 9. Plan

Ordered by value-per-risk. Steps 1–3 are committed; the rest are sequenced, not
promised.

### Phase 0 — Ambience (no new art, no pipeline, fully reversible)
1. Wrap `InteractiveViewer` in a `Stack` so a screen-space layer has somewhere to live.
2. Cloud-shadow layer (world space) + weather particle layer (screen space, pan-lagged).
3. `ambience` block in `map_style`, defaulted per template.
- **Acceptance:** Corvane feels alive at rest; steady 60fps on a real device; no
  regression in seating or hit-testing.
- **Why first:** it answers empirically how much of "premium" was motion rather
  than geometry, before a single model is built.

### Phase 1 — Declare the camera  ← STARTED
4. Add `camera` to `map_style`; back-fill Iron Verdict's by measurement.
5. Record it in the plate prompt template for future genre packs.
- **Acceptance:** a written camera spec that Blender and the placement engine both consume.
- **Method (works, use it):** under an orthographic camera a horizontal circle
  projects to an ellipse whose squash is exactly `sin(elevation)`. Round tower
  tops are circles, so elevation is *measured*, not guessed. Render the same
  subject through `blender/render_asset.py` at a sweep of elevations and match
  the ellipse.
- **First reading: ~40–45°, not the 58° assumed.** ⚠️ That reading is off the
  *sprite*, which is only a proxy — the plate is the real target, and the fact
  that the shipped gate still needed a −4° roll says sprite and plate cameras are
  close but not identical. Measure a circular feature on the PLATE before
  locking the number.

### Phase 2 — Prove the Blender loop on one asset
6. Headless Blender scaffold + render script targeting the declared camera.
7. **Rebuild Marblegate as the pilot** — we know exactly what correct looks like,
   and it is the asset that beat us.
8. Compare against `sprite-marblegate-v5.webp` in the offline previewer
   (`everlore-server/scripts/preview-placement.py`).
- **Acceptance:** the rendered gate seats in the wall gap with **no** hand-tuned
  rotation, and has a real contact shadow.
- ⚠️ **The risk this phase exists to test: style drift.** Matching a Blender
  render to that plate's palette and light is real work. If it is off, a rendered
  gate on a painted city looks *worse* than today's sprite. One asset, proven,
  before committing to 27.

### Phase 3 — Kit of parts + the 12 archetypes
9. ~50 modular medieval pieces; the 12 archetypes assembled from them.
10. Re-render into the existing pipeline. No app change.

### Phase 4 — Rotate-on-select
11. 36-frame turntable output; scrub-on-drag in `_SelectionCard`.

### Phase 5 — Worlds leave TypeScript
12. Authored worlds become documents + an authoring tool, with the version bump automated.
13. Promote `check-interactive-plate.ts` into a placement engine.

### Phase 6 — Real-time 3D
Only if something genuinely needs it — free camera control, or avatars walking
the map. Assets from Phase 2–3 already export to glTF, so nothing is wasted.

---

## Rejected options (and why, so they are not re-proposed blind)

- **Real-time 3D terrain mesh.** The painted plates are carrying the visual
  quality; matching them in real time needs an art pipeline we do not have, and
  Flutter's 3D story is thin. Revisit at Phase 6, or if worlds become
  generated-per-player at volume — at which point the art-supply argument
  inverts and this should be re-litigated properly rather than half-remembered.
- **Image-to-3D as the primary asset path.** Style consistency across generated
  meshes is much harder to hold than across generated images. Available for
  organic one-offs with human curation.
- **Rotating a PNG to fix a camera mismatch.** Tried; failed four times. In-plane
  rotation corrects *alignment* only — a sprite drawn at the wrong elevation is
  wrong in three dimensions and needs new art. The distinction is recorded on
  `InteractiveSpriteDoc.rotation`; keep the two failures separate.
- **Per-landmark pan parallax.** Would unseat every placed landmark.

## Open questions

- Does the isekai template share the medieval kit, or need its own? (Assume its
  own; the reuse argument is *within* a template.)
- Turntable frame count and resolution — 36 is a guess, not measured.
- Whether plates should also emit a water mask at generation time rather than
  by post-hoc segmentation.

---

## Phase 1 finding — the plate is PERSPECTIVE, not orthographic (Sept 6 2026)

Measured on `plate_central.webp` itself, not on a sprite proxy. Round torch
platforms on the city wall are circles on the ground plane, so their squash
gives the local elevation directly:

| feature | plate y | ellipse squash | elevation |
|---|---|---|---|
| wall torch, upper wall | ~965 | 0.60 | ~37 deg |
| wall torch, lower wall | ~1070 | 0.44 | ~26 deg |

The elevation **changes across the plate**. That is only possible under a
perspective camera close to the scene. Both readings are also far below the
58 deg previously assumed, and below the ~40-45 deg read off the sprite.

Consequences, and they are the whole reason this matters:

1. **The orthographic camera in `blender/render_asset.py` is wrong.** Under
   ortho one render is valid anywhere on the plate. Here the correct render
   depends on WHERE the landmark sits: a gate at the bottom of the plate needs
   a low, strongly-converging view; the same gate higher up needs a flatter
   one.

2. **This is why AI regeneration kept failing and cannot be fixed by prompting.**
   There is no prompt for "elevation 26 degrees at this exact position". In
   Blender it is not a parameter to guess at all - put the model at the matching
   position in a scene with the matched camera and the projection is correct by
   construction.

3. **The flat 0.16 sprite fraction is wrong in principle.** Under perspective,
   equal-sized buildings do not occupy equal image width. Second-order today
   (every landmark carries a hand-tuned `scale` that absorbs it) but it is the
   reason those scales are all different.

Method note: automated wall-slope detection failed for the third time - the
brightness threshold latches onto paving, rooftops and buttress faces rather
than the wall foot, and returns a different angle for every x-range chosen. It
is abandoned. Circular ground features measured at magnification are the only
reading that has held up.

Also measured, and useful for any future landmark on this wall: the wall does
not run straight through the gate. Its crest descends ~16 deg to the right of
the gap and ~7.5 deg to the left of it. The wall CHANGES BEARING at Marblegate,
so no single rigid sprite can be parallel to both sides. `rotation` there is a
compromise between two angles, never a match.

**Next**: fit the perspective camera by search, not analysis - build the scene
in Blender, drop a proxy box where the gate goes, sweep fov/height/tilt, and
score each render against the plate crop. The plate is the ground truth; a
closed-form fit off three hand-read ellipses is not trustworthy enough to
commit 27 assets to.

### The dominant defect is DETAIL FREQUENCY, not camera (Sept 6 2026)

Crop a wall tower from the plate and a tower from `sprite_marblegate`, scale
both to the same on-screen size, and put them side by side. The plate resolves
NO individual stones - it is soft, hazy, warm, low-contrast, painted at a scale
where a tower is ~60px and masonry is implied rather than drawn. The sprite
draws every single block, with hard edges, high local contrast and cool
grey-green stone.

Composited, the sprite carries roughly an order of magnitude more detail per
screen pixel than the ground it stands on. That is what reads as "sticker",
and it is why the eye rejects it before noticing anything about the camera.
Ranked by how much each contributes:

1. detail frequency  - sprite draws masonry the plate only implies
2. colour temperature - plate is warm sandstone, sprite is cool grey-green
3. camera elevation   - see above
4. no contact shadow  - the landmark does not touch its ground

**This is a warning for the Blender route, not an argument for it.** A default
Blender render is CRISP. Rendered naively it will sit even further outside the
plate than the AI sprite does. Matching requires deliberately rendering DOWN to
the plate's detail frequency and colour temperature - low-frequency materials,
soft sun, warm grade, and a resolution that does not resolve masonry.

The upside is that this is now a measurable target instead of a taste
judgement: crop plate and render, match their radially-averaged spatial
frequency spectra and mean hue/temperature. That is the acceptance test for the
Marblegate pilot, alongside "seats with no hand-tuned rotation".

Scale caveat, stated honestly: at the map's OPENING zoom the gate draws ~145px
wide and reads acceptably. The defect is obvious the moment the player pinches
in, which they can do freely - so "fine until you look at it" is not an answer
for a premium surface.

---

# DECISION (Sept 6 2026): landmark sprites are GONE. Markers replace them.

Shipped this session. Superseded everything above about seating, matting and
camera-matching landmark art - that whole problem class no longer exists.

## What changed and why

A composited landmark claims to be PART of the painting, so it has to match
camera, detail frequency, colour temperature and light all at once. We never had
more than two right. The decisive measurement: crop a wall tower from the plate
and a tower from a sprite, scale both to the same on-screen size, and the plate
resolves no individual stones while the sprite draws every block. The sprite
carries roughly an order of magnitude more detail per screen pixel than the
ground it stands on. THAT is what reads as a sticker - ranked above camera
elevation and above the missing contact shadow.

A marker does not claim to be part of the painting. It is a symbol drawn over
it, so it has to match nothing. There is no failure mode left to fix.

Tried and rejected: a small carved marker STONE instead of an abstract pin. At
map scale it is 11x18px, reads as a smudge, and is just a small sprite - it
inherits the exact problem being deleted. The clean ring works BECAUSE it is
abstract. The typography carries the whole design; Didot's hairlines break up
over busy terrain, Georgia Bold 14px with letterspacing and a dark halo holds
over paving, roofs and forest alike.

## Consequences

- ~1000 planned landmark assets (27 per world x 40 worlds) become ~6 marker
  variants per template, reused across all twenty worlds.
- Authoring a place goes from hours (generate, mat, seat, tune) to minutes
  (pick a point, type a name).
- The chroma-key matting stage, the alpha-mask hit test, sprite rotation and the
  camera-matching problem are all dead code paths.
- The PLATE must now paint its places, because nothing else draws them.

## Sprite art is UNREFERENCED, not deleted

Revision 5 objects stay in S3 untouched. Unreferencing gets 100% of the benefit
(nothing ships, nothing downloads) at zero risk, storage is a few MB, and
deleting would burn the fallback plus break CloudFront inconsistently while
cached edges still serve what the origin lost. Revisit only after playtest.

## Sealed vs rumoured: two signals, not two shades of dim

- **rumoured** -> mist. The place is not known, so it is not visible. Existing
  `texture_fog` does this.
- **sealed** -> NOT darkened. Darkening reads as disabled UI. A sealed place is
  KNOWN and that is the whole point - the player must see where they cannot go.
  Plate stays at full brightness; the MARKER goes cold iron instead of gold and
  carries a lock, with the authored `sealed_reason` on tap.

## Shipped

- `IRON_VERDICT_REVISION` 5 -> 6, `WORLD_VERSION` -> 24.
- Every location `map_render: 'marker'`. `sealed_marker_asset_id` removed.
- `IRON_VERDICT_ASSETS` no longer emits sprites - plates, scenes and textures only.
- plate_vaskir REGENERATED via the Codex app and published. The old plate had two
  cabins for seven authored places; the new one paints all seven. Anchors
  re-pointed to the art and verified by contact sheet.
- Flutter needed NO change: the marker path already existed for `map_render:
  'marker'` (constant-size hit target, inverse-scaled pin, lock, label) and was
  simply unreachable because `CITY_PLATE_LOCATIONS` was empty.
- Verified on the Android emulator.

## `check-interactive-plate.ts` rewritten - and it deliberately has NO verdict

The old script counted open plazas big enough to stand a sprite in; under
markers that is backwards. The replacement was first written to score "is
something built under this anchor" and it confidently failed all seven places
that are plainly visible in the art. Two reasons worth keeping:

- **Edge energy INVERTS on natural terrain.** A snow plate is dense rock,
  conifer and drift texture everywhere, so the median is high and a large smooth
  longhall roof scores BELOW it and reads as empty ground.
- **Colour reads shadow as water.** Shadowed snow is blue-dominant and dark, so
  a mountain pass measured 77% "water".

That is the third pixel-statistic detector written for this map that produced a
confident wrong answer (the wall-slope one failed twice before it). The script
now emits EVIDENCE plus a contact sheet of every anchor and passes judgement on
nothing. No threshold in it gates anything.

## Two defects found while shipping this, both silent

- The asset pipeline classifies by FILENAME PREFIX, so working files named
  `plate_vaskir_v2.png` and the audit's own `plate_*.anchors.png` were published
  AS PLATES and silently lengthened the stitched map to 1122x6825 with seven
  bands. The audit sheet now writes to /tmp, never beside the source.
- `iron-verdict-dimensions.ts` is generated with UNQUOTED keys, so an asset whose
  name carries a dot emitted invalid TypeScript - and because the pipeline
  IMPORTS that file, it could no longer run to fix itself. Keys are now
  JSON.stringify'd.

## Ambience: Flutter fragment shaders, NOT Blender

Reversing the earlier plan. Snow, mist and cloud shadow should be GLSL
`FragmentProgram` - zero download size, and they respond to zoom, which a
pre-rendered loop cannot (zoom in and the snowflakes become blurry rectangles;
it also visibly repeats). The river needs a flow mask per plate plus a scrolling
distortion shader; the mask is extractable offline but MUST be verified by eye
given the record above. Blender's only surviving use is turntable renders for
rotate-on-select, which composite against nothing - a later polish item, not
the foundation.

## All five plates regenerated (Sept 6 2026, revision 8, WORLD_VERSION 26)

| plate | places | shape |
|---|---|---|
| vaskir | 7 | square |
| north | 6 | 3:2 landscape |
| central | 8 | 4:5 portrait |
| south | 2 | 3:2 landscape |
| serevane | 7 | square |

30 anchors, every one verified against a contact sheet. Generated through the
Codex APP driven in the BACKGROUND (`app_click` / `app_type`, no screen
takeover). The `codex` CLI cannot be used - 0.136.0 is older than the app and
rejects its model.

What makes a prompt work, distilled from five runs:

- Give the reference file path and say "match its camera, palette, light
  direction and finish precisely, because it sits in one continuous map".
- State the aspect ratio explicitly and numerically ("LANDSCAPE 3:2, exactly 1.5
  times wider than tall"). Every run honoured it.
- Say what the reference gets WRONG ("almost empty, showing only two cabins").
- Describe each place as a PHYSICAL BUILDING, never by its story name. "A ring
  of tall standing stones on a flat cleared snowfield" works; "Hjarnmoot" does
  not.
- Say exact positions do not matter. Placement precision is not reliable, and
  under markers it is not needed - re-point the anchor to the art instead. This
  is the single biggest reason the marker design is cheap to author.
- Forbid text, labels, icons and borders explicitly. Every run obeyed.

⚠️ Marblegate is now PAINTED INTO THE WALL by the plate. The two days spent
seating a gate sprite in that gap - the rotation, the scale, the camera
measurement - produced nothing that survived. The `scale`/`rotation` hack on
that row is deleted.

## Fog: darkened -> veiled

`_unexplored` was `0xC2090B10`, a near-black scrim at 76%, and the cloud texture
was MULTIPLIED by a dark navy - so the mist was darker than the ground it
covered, which is not what mist does. Over a bright snow plate it read as an
underexposed image and buried seven painted settlements.

Now `0x80BAB2A8`, a pale warm veil, with the texture tinted `0xFFD8CFC2`. The
land stays legible and the world keeps its gold. A cold grey-blue `0x85A8B2C4`
is a one-constant swap for a colder genre pack.

⚠️ The painter draws OVER the plates, it does not filter them, so it cannot
desaturate. An early mock that desaturated is not achievable in this widget -
judge any fog proposal with a veil-only mock.

---

# Worlds are DATA now (Sept 6 2026, revision 9, WORLD_VERSION 27)

`src/worlds/iron-verdict.ts` is DELETED. `src/worlds/` holds one generic module
and a data directory:

    src/worlds/world-source.ts              loader + CDN key builder (mechanism)
    src/worlds/data/index.json              which worlds exist
    src/worlds/data/iron-verdict.json       the authored world
    src/worlds/data/iron-verdict.dimensions.json   generated by the pipeline

The JSON is read with `readFileSync`, NOT imported. A static import would bake
the content into the build and quietly reintroduce the exact problem the change
removes — editing a world would still need a rebuild.

`loadWorld(key)` returns null for an unknown world so the caller decides what
that means (404 to a player, usage error to a script); `requireWorld` throws.
Asset keys are built at load time from the world's `revision`, so authored data
never stores a CDN key and a revision bump touches one number.

Also gone: `interactive-world.service.ts` had `if (worldKey !== WORLD_KEY) throw
404`, which was the last place a world had to be NAMED IN CODE to exist. Any
world with a data file is now servable.

The generated dimensions file moved from `.ts` to `.json` for a concrete reason:
it was TypeScript that the pipeline both WROTE and IMPORTED, so one asset name
containing a dot emitted an unquoted key, broke the module, and left the
pipeline unable to run and repair its own output.

Four consumers were rewired: the service and three scripts. `sprite.variants`
(8 of them) were stripped from the data — they swapped landmark ART on a flag,
and there is no landmark art.

## Dead sprite code removed from Flutter

`world_map_view.dart` 783 -> 481 lines. Deleted: `_AlphaMask` (decoded a
sprite's alpha channel so a tap fell through gaps in a silhouette),
`_ensureMask`/`_masks`, `_unrotate`, the whole sprite branch of `_Landmark`, and
`rotationDegrees` end to end including the domain model. `_rectFor` is now
markers only. `_Landmark.build` is one line.

⚠️ Salutary: this file is UNTRACKED in git, so a bad scripted edit had no
version to restore from. A `rindex('}')` truncation and two blanket `replace`
calls that matched the wrong class cost a reconstruction. Read the class layout
before scripting edits across a file with several classes in it.

## Serevane regraded

The first Serevane came back bright turquoise and did not sit with the other
four plates. Re-prompted with "keep the composition and all seven places
EXACTLY, change only colour and light", naming plate_central and plate_south as
the grade reference. Composition held perfectly — all seven anchors still landed
with no re-pointing. Regrade-in-place is a reliable second pass.
