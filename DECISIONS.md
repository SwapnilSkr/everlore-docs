# Decisions

Settled product and engineering calls, so they are not re-asked. Each entry says what was decided, why, and what would reopen it. Newest first.

---

## Voiced lines, scene images and talking-character video in chat worlds  ·  Oct 15 2026

**Decided (owner):** chat worlds get three optional media layers, all through OpenRouter on our one key. Unlike the authored world art below, these are generated **while playing**, at the player's request. That is a new use, not a reopening of the Codex-app decision.

- **Voice.** Only the quoted lines a character speaks are voiced, never the narration.
  - Each spoken line gets a play button; the player taps to hear it. Auto-play may come later as a profile switch.
  - **Model: ElevenLabs v4 Turbo** (owner, Oct 16 2026, after listening to the probe): the best acting and cleanest delivery. At $20 per million characters it costs ≈ ₹0.17 per line. Fish Drama 3 was 10× cheaper but less good, and its launch price may be a promotion. It is the fallback to revisit.
  - Each character gets one voice the first time they speak, chosen from the model's 21 built-in voices by a deterministic fit (no model call), then kept. There is no automatic fallback model, because it would change the character's voice.
  - Feeling comes from how the line is written plus at most one performed tag: `[shouting]`, `[whispers]`, `[sighs]` or `[breathless]`, the four the probe proved are performed, not spoken. Genlife proved a written style note is read aloud on OpenRouter.
  - Intimate lines are voiced only for players who opted in to mature themes in their profile.
- **Images.** A "Picture this" button on a turn.
- **Video.** A "See them say it" button on a spoken line. It is generated in the background and arrives in the chat.
  - **A scene cut, not only a talking head** (owner, Oct 16 2026): an opening shot from the narration, then the character saying the line, then an optional closing shot. About 10–12 s.
  - The character's portrait is lip-synced to the voice line they already heard (never an invented voice).
  - Scene shots avoid face close-ups, because video models drift on faces. Estimated ≈ $0.45–0.65 (₹38–55) a video against $0.30 for the talking head alone. The cap drops from 3 to 2 a day to hold the cost.
- **Portraits, one per character, separate from the cover.** Covers are never animated, edited or used as a face. Each character gets its own head-and-shoulders portrait, used to lip-sync videos and as the face reference in "Picture this" scenes.
  - **A portrait belongs where the character belongs.** The world's cast (and a character world's companion) has a portrait **on the world**, made once and shared by every save and player. A character born during play has a portrait **on that save's card only**; the same name in two saves is two people.
  - **Made only on a tap, then kept.** A world cast portrait is made the first time any player taps for that character. Our launch worlds get theirs painted in the Codex app with the covers. Every quoted line can be heard (owner, Oct 16 2026). **Media never creates a character card**: who gets a card stays the story pipeline's call (codex, death, extraction, location graph untouched). A carded speaker uses the card's voice and portrait. A speaker with no card gets a voice picked from the prose's hints, the same for the same handle within a save, stored on the line, never on a card; they're pictured from the prose and filmed with a **passing face**: painted from the prose on the first video that needs it, kept per save like their voice, never on a card and never canon. Any turn can also be filmed without dialogue ("Film this": an opening and closing shot, one video). A child is voiced with the lightest adult voice; an unclear gender never refuses. When the codex merges two cards, the voice, portrait and look carry to the survivor.
  - **An unknown look is filled in and becomes canon.** At the first tap, a concrete look consistent with everything known (world/card `appearance`, prose, role, style) is written to the card's `appearance`. The narrator already reads that field, so the story then describes the person in the picture.
  - **Lasting vs passing looks.** Face, hair, build and permanent marks are lasting: they update that save's card. Clothes, dirt, blood and costume are passing: they go into one picture's prompt and never change the portrait. A lasting change counts only with a quoted sentence from the turn as evidence, never a model's guess.
  - **A change never generates on its own.** It updates the card text at once and marks the portrait out of date. The repaint happens only on the next tap that needs the face, using the old portrait as reference so it stays the same person. A story-caused repaint is free (within the ~20 portraits/day cap). A repaint the player asks for counts as one image. A player repainting a world character overrides it for that save only.
  - **The player's own character** is never voiced or animated, but can appear in pictures. Until they have a face, pictures are framed through their eyes or over the shoulder. A "Give yourself a face" offer paints one from the world protagonist's `appearance` or the persona's description. A protagonist's portrait starts on the world, like the cast. A persona's lives on the persona and follows it across worlds; lasting changes stay on the save.
  - **Creator worlds.** Portraits made on a player's tap need no review: they come from already-approved text and pass the image safety check. A creator repainting a cast face on a public world is a meaningful edit and goes back to review, like a cover change.
  - **No uploaded photo is ever a face**: not a cover, not a selfie, not a persona. Every image records its source (`generated`, `authored`, `uploaded`), and only `generated` and `authored` faces may be animated or used as references. Older images with no record count as `uploaded`.
  - The first portrait is free (not counted against images), capped at ~20 new portraits per player per day.
- **Images and video may be suggestive, never explicit** (owner, Oct 16 2026; this replaces "always SFW"). Romance, sensual mood, a kiss, an embrace, implied intimacy and revealing-but-covered clothing are allowed for every player. Full nudity and sexual acts are blocked for everyone, opted in or not. Anyone who looks underage is never shown in a sensual or suggestive way, and graphic gore stays blocked. Mature narration itself still follows the opt-in. The rule is enforced structurally, not only by prompt:
  - the visual description is safety-checked before generation, and the result is checked before it is shown;
  - a rejected moment shows "This moment stays unseen", is not charged and does not count.
- **Limits.** Separate daily counts for voice, images and video, reset at UTC midnight like the turn cap, the same for every tier during the promo. Starting numbers: 100 voiced lines (owner, Oct 16 2026: a heavy player then costs at most ≈ $6 / ₹510 a month in voice), 15 images, 2 videos (owner, Oct 16 2026, with the scene cut). Hitting one shows a calm in-world line, not an error.
- **Triggers.** Only the player's tap starts generation; the story never generates media on its own.

**Why:** tap-to-generate means we never pay for media nobody asked for and keeps the caps fair. A talking character keeps the face and voice consistent with what the player already heard. Mature voice follows the existing opt-in, while pictures and video allow suggestive but never explicit content for everyone.

**Models (owner, Oct 16 2026, after a paid probe, `.agents/reports/media-probe.md`, ≈ $0.73 / ₹62):**
- **Images:** Flux 3 Klein, ≈ ₹0.77 per image (owner, Oct 16 2026). It replaced Flux 2 Klein (≈ ₹1.30), whose own filter refused ordinary scarred faces as "graphic violence". Flux 3 Klein accepted them, matched the painted house style best of six models tested side by side, and draws anime well. It returns JPEG.
- **Scene shots (the scene cut):** Grok Imagine Video 1.5 Lite, with Wan 3.0 as the automatic fallback (owner, Oct 16 2026, after the scene probe, ≈ $0.64 / ₹55). A full cut costs ≈ $0.40–0.44 (₹34–38). Each shot starts from its own wide still, never a face close-up.
- **Video:** HeyGen Avatar IV, ≈ ₹25 per 6 s clip. It lip-syncs the portrait to our exact recording and keeps the painted face. Seedance swapped in a different voice and drifted to a photoreal face; Veo invents its own voice.
- No model refused the intimate voice lines.
- **Art style follows the world** (owner, Oct 16 2026). Media is drawn in the style the world's locked narrative voice maps to, the same table the cover uses, with mood words left out of the drawing style. There is no separate authored art-style field for now. The player's own face is kept per style, painted on first tap in a world of that style. A portrait painted in an older style repaints on the next tap.

**Reopen when** costs per player run over the caps, or a tier split is wanted after the promo.

---

## Launch catalog: our chat worlds go straight to prod, covers from the Codex app  ·  Oct 15 2026

**Decided (owner):** the ~40 launch chat worlds are created **directly on prod**, with no staging step. Prod has no real users yet. Covers come from the Codex desktop app (`WORLD_CREATION_PLAYBOOK.md` §4B), not the image API.

**How:**
- Worlds are authored as files and written by an idempotent seed (keyed by slug), the same "worlds are data" pattern as walks.
- The seed runs locally against dev first, then on `everlore-prod` (EC2 `i-0c113fabad76aecf4`) through SSM with the box's own env.
- Worlds are published under one official account named **Everlore**, not the owner's personal one.

**Reopen when** prod has real players. From then on, new catalog worlds go through dev first and a reviewed prod run.

---

## Walkable worlds guide the player like Assassin's Creed, and reward them with achievements  ·  Oct 4 2026

**Decided (owner, approved mockup):**
- **One tracked goal** is always shown top-left: quest title, goal, and where it is.
  - "Goal complete" and "New goal" appear as centred moments.
  - Tapping the tracker opens the goals journal: tracking, other leads (which can be pinned), and done.
- **Achievements** arrive as a slide-up card and are listed in the in-world clock sheet ("The road behind you"), as a second tab.
- The conversation shows both figures, your last words, one reply, topic chips, and a row with "Say something", history and step away. There is no Next card.
- Every world's prose is written in plain modern English.

**How:**
- Goals and achievements are authored per world: `<world>.goals.json` and `<world>.achievements.json`.
- Goals are derived on every read. The tracker follows what the last deed opened, then distance.
- Achievements are a record of the *player*. They are stored in `interactive_achievements` and survive rewind and walk deletion. Only account deletion or world deletion removes them.
- Spec: `phases/phase-1.md` § W10.

**Reopen when** a world needs branching quest logic that a goal-by-choice list cannot express.

---

## World art is made in the Codex app and seeded from files — no API image generation  ·  Oct 3 2026

**Decided (owner):** our own worlds' art (map plates, scene paintings, portraits) keeps coming from the Codex desktop app, and worlds keep being authored as files under `everlore-server/scripts/fixtures/walks/` that a seed writes into Mongo. We do **not** build an API image-generation pipeline (OpenRouter or otherwise) for now.

**Why:**
- **Cost.** Image generation is included in the ChatGPT Pro plan we already pay for, so the Codex app costs nothing extra. One world the size of The Iron Verdict is 58 images (5 plates, 1 texture, 20 scenes, 32 portraits); at ~3 attempts per kept image that is ~175 generations — roughly **$7 (₹600)** on a cheap API model and **$35 (₹3,000)** on a top one, per world (Oct 2026 ballpark; prices move).
- **Quality.** API image models return no real alpha (they paint a fake checkerboard), so cut-outs need flat `#00B140` chroma green and keying either way; the Codex app has been the most consistent at that and at matching the kept house style. Failed attempts (a portrait on black instead of green) are a total loss and would each be billed.

**How it works today:** art → Codex app (AGENTS.md §5) → drop folder → `bun run assets:process -- --upload` → S3/CloudFront. World data → JSON files → `seed-walk-definitions` → the `interactive_worlds` document. **The game never reads the JSON at runtime**; the server loads every world from Mongo.

**Reopen only when** there is a real requirement and bandwidth for **creator or admin world authoring with generated art** — players or admins making their own worlds in the app, which the hand-driven Codex app cannot serve. That work would need: an API image model chosen on current prices, a per-world spend cap, probably a premium-only gate, chroma-key matting on the server, and DB-native authoring (admin console) instead of files.

---

## One AI bill: OpenRouter for every chat/structured call; embeddings stay on OpenAI  ·  Oct 2 2026

**Decided (owner):** every chat and structured-output model call goes through OpenRouter (bare OpenAI ids map to `openai/…`). **Memory embeddings** (`text-embedding-3-small` → Pinecone) stay on OpenAI-direct — too crucial to risk. New model choices default to OpenRouter ids.

**Reopen when** the owner wants embeddings moved too; that needs a parity test on recall first.

---

## Which model speaks  ·  Oct 3 2026

**Decided (owner):** chat worlds keep the DeepSeek narrator (`NARRATION_SFW_MODEL`). Walkable-world characters speak with Luna 6 (`CHARACTER_SPEECH_MODEL`, `openai/gpt-6-luna`) at `WORLD_SPEECH_REASONING=low` (~1.2 s to the first word, measured over 21 live lines). Duels keep Luna 6 at `CHARACTER_SPEECH_REASONING=medium`.

**Why:** walkable speech streams like a chat turn (`phases/phase-1.md` § W7). In live replays the narrator was fastest (~0.5 s) but thinner and repeated itself; Luna 6 medium was best-written but ~2.7 s; Luna 6 low kept the quality at ~1.2 s.

**Reopen when** a cheaper or faster model beats Luna 6 low on a live replay (`bun run walk:playthrough`), measured the same way.
