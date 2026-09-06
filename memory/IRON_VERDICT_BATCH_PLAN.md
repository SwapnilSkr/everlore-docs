# Iron Verdict — generation batch plan

96 images in **17 batches**. One batch per chat session. Do not attempt more
than one batch in a session — quality degrades and generation stalls.

## Instructions for the generating agent

Read this whole section before starting. It is written to be picked up cold by
a fresh session with no memory of the previous batch.

1. Open `IRON_VERDICT_TIER_A_PROMPTS.md` (batches 1–3, partly) and
   `IRON_VERDICT_ALL_PROMPTS.md` (everything else). Every prompt in those files
   is **complete and self-contained**. Paste it exactly as written. Do not
   summarise it, do not "improve" it, do not merge prompts, and do not drop the
   negative clauses — they are the only thing keeping the art on style.
2. Find the batch below whose checkbox is unticked and has the lowest number.
   That is your batch. Generate **only** the assets listed in it.
3. Save each image as `<asset_name>.png` — the exact name in the prompt heading,
   lowercase, underscores, no prefix, no suffix, no batch number. The processing
   pipeline matches on these names and skips anything it cannot identify.
4. Save every file at **full resolution, as emitted, uncompressed PNG**. Do not
   resize, do not convert, do not optimise. Matting happens before downscaling;
   a pre-shrunk file gets a green fringe that cannot be removed later.
5. Save every file into this exact folder, flat, with no subdirectories:

   ```
   ~/Desktop/sample files/sample files for everlore/world assets
   ```

   Create it if it is missing. Do not invent a per-batch subfolder — the
   pipeline reads one flat directory and every batch lands in the same one.
6. When every asset in the batch exists on disk, tick that batch's checkbox in
   this file and write the date beside it. Then stop and report which batch is
   next. Do not start it.

### Quality gate — reject and regenerate if any of these are true

- **Sprites and portraits:** the background is not flat green. If it shows a
  grey-and-white checkerboard, the model tried to fake transparency — regenerate
  and check the prompt still says chroma green. A checkerboard is a total loss;
  it cannot be matted.
- **Sprites:** a shadow, patch of ground, plinth or second building appears on
  the green.
- **Scenes:** a person appears in the foreground or midground. Distant crowds
  are correct and wanted; a figure standing on the empty ground is not, because
  a character cutout gets composited exactly there.
- **Anything:** visible text, lettering, runes, numbers, a signature, a border
  or a UI frame.
- **Anything:** lava, magma, glowing orange cracks in the ground, wildfire, or
  a burned-out wasteland. This world is inhabited and prosperous.

### Plates are already done — the socket rule is for the next world

All five plates landed in batch 1, so no remaining batch generates one. The
socket rule (a plate must leave empty plazas for the landmark sprites) is
recorded in `IRON_VERDICT_WORLD_BIBLE.md` §6 and applies to the next genre
pack's plate prompts, not to anything below.

### Style anchor

The first batch generated becomes the visual reference for all later ones. When
starting a later batch, look at `plate_central.png` and `sprite_verdict_ring.png`
first and match their palette, light direction and level of finish.

---

## Batches

### Terrain — 5

- [x] **Batch 1 — Plates** (5) · Tier A sheet §1–3, plus ALL sheet "Plates" · 2026-09-06
      `plate_north` `plate_central` `plate_south` `plate_vaskir` `plate_serevane`

### Sprites — 25

- [x] **Batch 2 — Hero sprites A** (6) · 2026-09-06
      `sprite_verdict_ring` `sprite_sundial_market` `sprite_hollowforge`
      `sprite_chainhouse` `sprite_marblegate` `sprite_sable_court`
- [x] **Batch 3 — Hero sprites B** (6) · 2026-09-06
      `sprite_iron_fane` `sprite_drowned_stair` `sprite_thornhollow`
      `sprite_blackreach_hold` `sprite_hjarnmoot` `sprite_gilt_exchange`
- [x] **Batch 4 — Archetype sprites A** (6) · 2026-09-06
      `sprite_arch_village` `sprite_arch_farmstead` `sprite_arch_watchtower`
      `sprite_arch_ruined_chapel` `sprite_arch_longhall` `sprite_arch_keep`
- [x] **Batch 5 — Archetype sprites B** (6) · 2026-09-06
      `sprite_arch_mine_head` `sprite_arch_quarry` `sprite_arch_wharf`
      `sprite_arch_manor` `sprite_arch_bridge` `sprite_arch_camp`
- [x] **Batch 6 — Map kit** (3) · 2026-09-06
      `sprite_sealed_marker` `sprite_ruin_marker` `texture_fog`

### Scenes — 22

- [ ] **Batch 7 — Scenes A, Corvane** (6)
      `scene_verdict_ring` `scene_sundial_market` `scene_hollowforge`
      `scene_chainhouse` `scene_marblegate` `scene_sable_court`
- [ ] **Batch 8 — Scenes B, Aldermere** (6)
      `scene_iron_fane` `scene_drowned_stair` `scene_ravensgate_bridge`
      `scene_thornhollow` `scene_greywither_abbey` `scene_blackreach_hold`
- [ ] **Batch 9 — Scenes C, Vaskir + Serevane** (6)
      `scene_rimegate` `scene_hjarnmoot` `scene_vaskhold`
      `scene_skarrow_longhall` `scene_hundred_docks` `scene_gilt_exchange`
- [ ] **Batch 10 — Scenes D + duel stages** (4)
      `scene_casa_verrine` `scene_whisper_baths`
      `duel_stage_oathcircle` `duel_stage_harbour`

### Characters — 30

- [ ] **Batch 11 — Portraits A, Aldermere nobility** (6)
      `portrait_aldric_morrow` `portrait_sereth_morrow` `portrait_barran_coyle`
      `portrait_arbiter_vell` `portrait_iselle_harrowfield` `portrait_hob_cardew`
- [ ] **Batch 12 — Portraits B, Aldermere common** (6)
      `portrait_wren` `portrait_gaunt` `portrait_sister_mave`
      `portrait_the_ferryman` `portrait_the_herald` `portrait_marek_sain`
- [ ] **Batch 13 — Portraits C, Vaskir + Serevane** (6)
      `portrait_hjalvar_skarrow` `portrait_brynn_skarrow`
      `portrait_oathspeaker_ranneke` `portrait_aurel_verrine`
      `portrait_ysolde_verrine` `portrait_sabra_idre`
- [ ] **Batch 14 — Expressions A** (6)
      `portrait_cassian_resolute` `portrait_cassian_broken`
      `portrait_nara_wary` `portrait_nara_warm`
      `portrait_sereth_pleading` `portrait_sereth_cold`
- [ ] **Batch 15 — Expressions B** (6)
      `portrait_iselle_furious` `portrait_iselle_grieving`
      `portrait_brynn_challenging` `portrait_brynn_sworn`
      `portrait_ysolde_alarmed` `portrait_ysolde_resolved`

### Variants — 12

- [ ] **Batch 16 — Awakened sprites** (8)
      `sprite_verdict_ring_awakened` `sprite_hollowforge_awakened`
      `sprite_thornhollow_awakened` `sprite_marblegate_sealed`
      `sprite_sable_court_mourning` `sprite_iron_fane_open`
      `sprite_drowned_stair_risen` `sprite_blackreach_hold_besieged`
      ⚠️ Open the matching base sprite before generating each of these. The
      silhouette, materials, palette and camera angle must be identical — only
      the *state* changes. A variant that looks like a different building is a
      reject.
- [ ] **Batch 17 — Time-of-day scenes** (4)
      `scene_verdict_ring_night` `scene_sundial_market_dawn`
      `scene_chainhouse_day` `scene_hundred_docks_storm`

---

## Order matters

Batches 1–3 first, always. They set the visual reference and they are the only
assets that block the renderer. Batches 16–17 last — they are polish, and they
are what gets cut if the budget runs out.

If a batch produces something you are unsure about, keep it and flag it. The
pipeline emits a contact sheet of every asset at the end, which is where style
drift becomes obvious. It is far cheaper to judge 96 thumbnails side by side
than to second-guess one image alone.
