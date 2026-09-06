# The Iron Verdict — Asset Prompt Sheet

Everything needed to art the interactive world as a **terrain plate + placed
sprites** scene graph (Model B). Lore, place names and reasoning:
[IRON_VERDICT_WORLD_BIBLE.md](./IRON_VERDICT_WORLD_BIBLE.md).

Generate → upload → hand back CDN URLs. Then I mat, stitch, place and wire.

**Supersedes `ASHEN_CIRCUIT_ASSET_PROMPTS.md`, which was wrong about the style.**

---

## 0. Rules that are not optional

Verified against OpenRouter on Sept 5 2026 with three separate models.

1. **Never write "transparent background", "alpha channel", or "PNG with
   transparency".** No image model on OpenRouter returns a real alpha channel —
   every one of them *paints a fake checkerboard* instead (measured: a flat
   229/255 grey grid). Sprites are generated on **flat chroma green `#00B140`**
   and cut server-side with `sharp`. That matting pass is mine, not yours.
2. **Prepend the STYLE BIBLE (§1) verbatim to every prompt.** Do not paraphrase
   it per asset. It is the only reason independently generated assets composite
   into one world. This is the highest-leverage line on the page.
3. **One camera for the whole map.** Plates *and* sprites are painted from a
   **three-quarter overhead view at roughly 45° elevation**, sun from the upper
   left. A sprite painted at eye level cannot be placed on a plate painted from
   above, and no amount of scaling fixes it.
4. **Models.** Building sprites and architecture → `openai/gpt-5.4-image-2`
   (crispest silhouettes, best heraldry, cleanest chroma edge). Full-bleed
   painterly scenes and terrain plates → `bytedance-seed/seedream-4.5`.
5. **Do not regenerate Cassian Vale or Nara Voss.** They are the reference the
   rest of the kit is matching.

---

## 1. STYLE BIBLE — prepend verbatim to §2–§6

> Detailed semi-realistic fantasy digital painting, cinematic concept art,
> rich material texture, painterly brushwork with photographic depth and
> volume. Late-Roman into early-medieval architecture: pale travertine and
> weathered limestone arcades, dark oak timber, slate and lead roofs, wrought
> iron, crimson and deep indigo heraldic banners with brass fittings. Lighting
> is a warm low golden key from the upper left with deep soft shadows falling
> to the lower right, plus amber torch, brazier and lantern glow. Palette: warm
> stone grey, bone, ochre, aged brass and gold, deep crimson, muted indigo,
> olive. Absolutely NO lava, NO magma, NO volcanic rock, NO glowing orange
> cracks in the ground, NO wildfire, NO ash wasteland, NO burning ruins. This
> is a living, inhabited, prosperous city and countryside — warm and lived-in,
> not a hellscape. No text, no lettering, no watermark, no UI, no logo, no
> border, no frame.

---

## 2. Terrain plates — 3 assets · TIER A

Three square plates stacked vertically, which I stitch into one tall
~2048×5800 map. That stack is where the "long, open world" feeling comes from.

**The seam trick:** each plate's top and bottom edge is a natural barrier — a
mountain ridge, a river, a cloud bank. Continuity across the seam then never
has to be pixel-exact, because there is nothing continuous to match. Do not
drop these edge instructions.

**Critical:** plates carry *generic anonymous background housing* for density
but must **NOT** contain an arena, a market hall, a forge, a barracks, a city
gate, a bridge or a camp. Those are sprites, placed on top. Plates must leave
**flat open clearings** for them to land in — an arena painted into the ground
*and* an arena sprite somewhere else is exactly today's bug.

Format: square, largest offered (2048×2048 preferred). JPEG fine.

### 2.1 `plate_north` — the uplands
> [STYLE BIBLE] Three-quarter overhead view from about 45 degrees elevation of
> northern upland country in a fantasy kingdom, painted as a rich campaign map.
> Dark pine forest, rocky highland, terraced barley fields, winding pale dirt
> roads, drystone walls, a gorge cut by a fast river. A scatter of tiny
> anonymous thatched cottages and farm hamlets for scale — no large landmark
> buildings, no arena, no market hall, no forge, no barracks, no gate, no
> bridge, no camp. Leave two flat empty clearings of open ground beside the
> road where buildings could later stand. The very top edge of the image is an
> impassable snow-capped mountain ridge running fully left to right; the very
> bottom edge is a wide slow river running fully left to right.

### 2.2 `plate_central` — Corvane, the walled city
> [STYLE BIBLE] Three-quarter overhead view from about 45 degrees elevation of
> a dense walled city district in a fantasy kingdom, painted as a rich campaign
> map. Tight clusters of small anonymous tiled-roof houses, narrow cobbled
> lanes, a broad paved processional avenue running north to south, courtyards
> with wells and washing lines, a canal with low stone footbridges, market
> awnings in crimson and ochre. The great curtain wall of the city crosses the
> lower third. No large landmark buildings at all — no arena, no colosseum, no
> market hall, no forge, no barracks, no gatehouse. Leave four flat empty paved
> plazas of open ground where large buildings could later stand, one of them
> notably larger than the rest. The very top edge is a wide slow river running
> fully left to right; the very bottom edge is dense dark woodland running
> fully left to right.

### 2.3 `plate_south` — the low country
> [STYLE BIBLE] Three-quarter overhead view from about 45 degrees elevation of
> low coastal marsh country in a fantasy kingdom, painted as a rich campaign
> map. Salt marsh, reed beds, tidal creeks, boardwalks on pilings, a wide river
> mouth opening to a pale sea along one edge, low grassy burial mounds, wind-
> bent trees. A few tiny anonymous fishing shacks and upturned boats for scale
> — no large landmark buildings, no temple, no keep, no village hall, no jetty
> structure. Leave three flat empty clearings of firm open ground where
> buildings could later stand. The very top edge is dense dark woodland running
> fully left to right; the very bottom edge is open pale sea running fully left
> to right.

---

## 3. Landmark sprites

Every sprite prompt is **[STYLE BIBLE] + [SPRITE BLOCK] + the subject line.**

### SPRITE BLOCK — paste into every prompt in §3 and §4

> A single isolated building rendered alone at the centre of a completely flat,
> uniform, saturated chroma-key green background, colour #00B140, filling the
> entire frame edge to edge behind and around the subject. The green is
> perfectly even with no gradient, no vignette, no texture, no pattern and no
> checkerboard. Nothing else is in the image — no ground, no grass, no terrain,
> no road, no base plinth, no cast shadow on the green, no other buildings.
> The building is seen in three-quarter overhead view from about 45 degrees
> elevation, lit by a warm low golden key from the upper left. Square image,
> the building filling roughly 80 percent of the frame.

### 3.1 `sprite_verdict_ring` — the colosseum · TIER A
> [STYLE BIBLE][SPRITE BLOCK] A grand oval Romanesque amphitheatre of pale
> travertine, three stacked tiers of arched arcades with engaged columns, a
> ring of tall crimson and indigo heraldic banners on iron poles around the
> upper rim, a raised awning of stretched canvas over one section of seating,
> two great arched entrance tunnels facing the viewer. Weathered, monumental,
> clearly the largest structure in a city.

### 3.2 `sprite_sundial_market` — TIER A
> [STYLE BIBLE][SPRITE BLOCK] A covered market hall: an open colonnaded stone
> pavilion with a low tiled roof, surrounded by a tight ring of merchant stalls
> under crimson, ochre and striped canvas awnings, hanging brass lanterns,
> barrels, crates and rolled carpets. At its centre rises a tall bronze sundial
> on a stepped stone pedestal, green with verdigris, its gnomon casting a long
> shadow. Busy and prosperous.

### 3.3 `sprite_hollowforge` — the blacksmith · TIER A
> [STYLE BIBLE][SPRITE BLOCK] A working blacksmith's forge: a squat stone
> building with a heavy slate roof and a tall soot-blackened brick chimney
> trailing a thin plume of grey smoke, one open side revealing a glowing orange
> forge fire and an anvil, a water trough, a grindstone, stacked iron bar
> stock, and a rack of finished blades and helms outside. Dark oak timber
> framing, iron strapping on the doors. The orange glow is confined strictly to
> the forge mouth and does not spill onto the ground.

### 3.4 `sprite_chainhouse` — fighters' quarters · TIER A
> [STYLE BIBLE][SPRITE BLOCK] A grim two-storey barracks block of heavy pale
> stone, small barred windows in a regular row, a single reinforced iron-banded
> door, an enclosed training yard along one side with a sand floor, wooden
> practice posts, weapon racks and a low colonnade of cells. Institutional,
> fortified, austere — a place where fighters are kept rather than housed.

### 3.5 `sprite_marblegate` — the city gate · TIER A
> [STYLE BIBLE][SPRITE BLOCK] A monumental fortified city gatehouse: twin
> massive drum towers of pale limestone flanking a deep arched passage with a
> raised iron portcullis, crenellated battlements, arrow slits, a pair of long
> crimson banners hanging down the front face, a short stub of thick curtain
> wall running out to either side and stopping cleanly. Weathered and heavily
> built.

### 3.6 `sprite_drowned_stair` — undercity mouth · TIER B
> [STYLE BIBLE][SPRITE BLOCK] The entrance to a flooded undercity: a broad
> stone stairway descending into dark still water, framed by a low Romanesque
> arch of pale weathered stone half-swallowed by the ground, an iron grate
> hanging open, mossy walls, a mooring ring and a single small flat-bottomed
> boat at the waterline. Damp, green-black stone. Only the arch and the top of
> the stair are visible above ground.

### 3.7 `sprite_ravensgate_bridge` — TIER B
> [STYLE BIBLE][SPRITE BLOCK] A high single-span stone toll bridge over a
> gorge: a steep arched roadway of pale weathered blocks with low parapets, a
> small square toll house with a slate roof and a lantern at one end, a lowered
> timber toll bar across the road, iron chains and a rope winch. The bridge is
> shown complete and isolated with nothing under it.

### 3.8 `sprite_thornhollow` — the camp · TIER B
> [STYLE BIBLE][SPRITE BLOCK] A fortified hidden camp in a hollow: a rough
> palisade of sharpened timber stakes wound with thorn brambles, a cluster of
> canvas and hide tents around a central firepit with a cook tripod, drying
> racks, a rickety timber watch platform, stacked shields and a single faded
> banner. Improvised but organised — a village in exile, not a robbers' den.

---

## 4. Shared map kit · TIER A

### 4.1 `sprite_sealed_marker`
Reused by every sealed location, including all eleven Chapter II places. This
one asset is what lets the map show a world far larger than we have art for.

> [STYLE BIBLE][SPRITE BLOCK] A single tall standing stone waymarker of pale
> weathered granite, roughly the height of two people, bound around its middle
> by heavy rusted iron chains fixed with a large closed iron padlock. A blank
> bronze plaque is set into its face with no writing on it. Cold blue-grey
> shadow, no warm light on the stone itself.

### 4.2 `texture_fog` — seamless
> [STYLE BIBLE, minus the architecture sentence] A seamless tileable texture of
> thick slow-drifting fog and low cloud, soft billowing volumes in cool grey
> and pale blue-white, painted on a pure flat black background so the fog reads
> as luminous smoke against darkness. Soft edges, no hard shapes, no
> silhouettes, no landscape, no horizon. The texture tiles seamlessly on all
> four edges with no visible seam.

---

## 5. Scene backgrounds — portrait 9:16

One per playable location. **Empty stages:** no characters, no people in the
foreground — cutouts are composited on top. Crowds in the *far background* are
wanted and are not "characters".

Composition contract, identical for all: horizon in the upper third, the lower
40 percent of the frame is open uncluttered ground, and the centre-bottom is
kept clear so a standing figure lands there without covering anything.

### 5.1 `scene_verdict_ring` — TIER A · **doubles as the duel stage**
This one asset serves both the location scene and the King's Choice combat
screen, so it is the most important image on the sheet. The crowd is the point.

> [STYLE BIBLE] Portrait 9:16 view standing on the raked sand floor of a grand
> Romanesque amphitheatre, camera low and looking slightly up across the arena.
> Sweeping tiers of pale travertine seating curve around and away, packed dense
> with a vast cheering crowd painted as small distant figures, bright sunlit
> canvas awnings in crimson and ochre stretched over sections of the stands,
> tall heraldic banners in crimson and indigo hanging between the arcade
> columns, a stone dignitaries' box with a carved balustrade. Warm late
> afternoon sun, drifting dust in the light, blue sky with high cloud. The
> lower foreground is bare raked sand with faint circular markings, completely
> empty of people. No figures in the foreground or midground — the crowd is
> distant background only.

### 5.2 `scene_sundial_market` — TIER A
> [STYLE BIBLE] Portrait 9:16 view standing in a busy medieval market square at
> dusk. Stone colonnade down one side, tight rows of stalls under crimson and
> striped awnings, hanging brass lanterns strung overhead casting warm pools of
> amber light, barrels, crates, rolled carpets and hanging herbs. A tall bronze
> sundial on a stepped pedestal stands mid-distance. Deep blue evening sky above
> warm lantern light. The lower foreground is open worn cobblestone, completely
> empty of people. Only faint distant silhouettes far down the colonnade.

### 5.3 `scene_hollowforge` — TIER A
> [STYLE BIBLE] Portrait 9:16 interior view of a working blacksmith's forge at
> night. A great stone hearth glowing amber down one side, an anvil on a
> blackened oak stump, hanging tongs and hammers, a quenching trough steaming,
> walls hung with finished blades, helms and mail, sparks drifting upward,
> heavy timber roof beams above. Warm firelight against deep brown shadow. The
> lower foreground is an open swept earth floor, completely empty of people.

### 5.4 `scene_chainhouse` — TIER A
> [STYLE BIBLE] Portrait 9:16 view of the sand training yard of a fighters'
> barracks at night. Heavy pale stone walls with small barred windows, a
> colonnade of dark cell openings along one side, wooden practice posts scarred
> with cuts, racks of blunt training weapons, iron braziers burning low and
> throwing long shadows across the sand. Cold moonlight above, warm brazier
> light below. The lower foreground is open raked sand, completely empty of
> people.

### 5.5 `scene_marblegate` — TIER A
> [STYLE BIBLE] Portrait 9:16 view standing in the deep arched passage of a
> monumental city gatehouse at night, looking out through the arch toward open
> country beyond. Massive pale limestone blocks, a raised iron portcullis
> overhead, iron-banded doors folded back, torches in wall brackets throwing
> warm light up the vaulting, long crimson banners, a customs table with
> ledgers and a lantern to one side. Deep blue night beyond the arch. The lower
> foreground is open worn flagstone, completely empty of people.

### 5.6 `scene_drowned_stair` — TIER B
> [STYLE BIBLE] Portrait 9:16 view of a flooded undercity chamber lit by a
> single shaft of daylight from a grate far above. Romanesque arches half
> submerged in still black water, wide stone steps descending into it, wet
> mossy green-black walls, iron rings and hanging chains, a small flat-bottomed
> boat moored at the step edge, drifting mist on the water. Cold pale light
> against deep green shadow, one warm lantern reflection. The lower foreground
> is a broad dry stone landing, completely empty of people.

### 5.7 `scene_ravensgate_bridge` — TIER B
> [STYLE BIBLE] Portrait 9:16 view standing at one end of a high stone toll
> bridge over a deep gorge at first light. The arched roadway runs away from
> the camera with low parapets, a small slate-roofed toll house and a lowered
> timber toll bar to one side, a lantern on a hook, mist filling the gorge
> below, pine ridges beyond. Cool dawn blue with a warm rim of sunrise along
> the ridgeline. The lower foreground is open worn roadway, completely empty of
> people.

### 5.8 `scene_thornhollow` — TIER B
> [STYLE BIBLE] Portrait 9:16 view inside a hidden camp in a wooded hollow at
> night. A ring of canvas and hide tents around a large central firepit, a
> sharpened timber palisade wound with thorn brambles behind, drying racks,
> stacked shields, a rickety watch platform in a tree, a single faded banner.
> Warm firelight against cold blue forest darkness, embers rising. The lower
> foreground is open packed earth, completely empty of people.

---

## 6. Optional · TIER C

Nothing here blocks Chapter I. Generate only if Tier A and B are done.

- `sprite_verdict_ring_awakened` — same prompt as §3.1 plus *"banners fully
  raised, torches lit along the upper rim, warm light spilling from both
  entrance tunnels"*. Swapped in when a Verdict is scheduled.
- `sprite_hollowforge_awakened` — §3.3 plus *"forge fire burning bright,
  thicker smoke from the chimney"*. Swapped in once the smith works for you.
- `sprite_thornhollow_awakened` — §3.8 plus *"banner raised, campfire built
  high, more tents"*. Swapped in after the pact.
- Scatter props for map density, each on chroma green per §3's SPRITE BLOCK:
  `prop_watchtower`, `prop_windmill`, `prop_shrine`, `prop_wreck`.

---

## 7. Delivery checklist

**Tier A — 15 assets, this is the floor for a playable Chapter I**

```
plate_north  plate_central  plate_south
sprite_verdict_ring  sprite_sundial_market  sprite_hollowforge
sprite_chainhouse    sprite_marblegate      sprite_sealed_marker
texture_fog
scene_verdict_ring   scene_sundial_market   scene_hollowforge
scene_chainhouse     scene_marblegate
```

**Tier B — 6 assets, completes the Chapter I graph**

```
sprite_drowned_stair  sprite_ravensgate_bridge  sprite_thornhollow
scene_drowned_stair   scene_ravensgate_bridge   scene_thornhollow
```

Hand back CDN URLs keyed by the asset name above. Then I:

1. mat the chroma-green sprites — key, despill, auto-trim, soft edge — resize
   and re-upload under durable versioned keys;
2. stitch the three plates with a feathered seam into one tall map;
3. author `map_position`, `anchor`, `scale` and `z` per sprite against the
   delivered plate;
4. derive hit geometry from each sprite's own alpha, so taps hit the true
   silhouette rather than a rectangle;
5. rewrite the renderer as one transformed map layer holding plate, sprites and
   fog — which also fixes the map not panning — with alpha-accurate hit
   testing, state-keyed sprite variants, a bounded camera, and fog reveal on
   discovery;
6. delete `_RoutePainter`; the connecting lines go, and the painted road on the
   plate becomes the visible route;
7. seed `iron-verdict` v1 and retire every `ashen-circuit` S3 object with
   `bun run retire:interactive-prototype-assets`.
