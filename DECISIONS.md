# Decisions

Settled product and engineering calls, so they are not re-asked. Each entry says what was decided, why, and what would reopen it. Newest first.

---

## Voiced lines, scene images and talking-character video in chat worlds  ·  Oct 15 2026

**Decided (owner):** chat worlds get three optional media layers, all through OpenRouter on our one key. Unlike the authored world art below, these are generated **while playing**, at the player's request. That is a new use, not a reopening of the Codex-app decision.

- **Voice.** Only the quoted lines a character speaks are voiced, never the narration.
  - Each spoken line gets a play button; the player taps to hear it. Auto-play may come later as a profile switch.
  - Each character gets one voice the first time they speak, chosen from the model's built-in voices by a deterministic fit (no model call), then kept.
  - Feeling comes from how the line is written plus 0–2 performed tags (`[whispers]`, `[laughs]`…). Genlife proved a written style note is read aloud on OpenRouter.
  - Intimate lines are voiced only for players who opted in to mature themes in their profile.
- **Images.** A "Picture this" button on a turn.
- **Video.** A "See them say it" button on a spoken line. The character's portrait is lip-synced to the voice line they already heard (a talking character, not an acted scene with an invented voice). It is generated in the background and arrives in the chat.
- **Images and video are always SFW**, enforced structurally, not only by prompt:
  - the visual description is safety-checked before generation, and the result is checked before it is shown;
  - a rejected moment shows "This moment stays unseen", is not charged and does not count.
  - Videos start only from portraits we generated, **never from uploaded photos**, so no real person can be animated.
- **Limits.** Separate daily counts for voice, images and video, reset at UTC midnight like the turn cap, the same for every tier during the promo. Starting numbers: 150 voiced lines, 15 images, 3 videos. Hitting one shows a calm in-world line, not an error.
- **Triggers.** Only the player's tap starts generation; the story never generates media on its own.

**Why:** tap-to-generate means we never pay for media nobody asked for and keeps the caps fair. A talking character keeps the face and voice consistent with what the player already heard. Mature voice follows the existing opt-in, while pictures stay SFW for everyone.

**Still open:** the voice model (Gemini 3.8 Flash TTS vs Fish Audio Drama 3 vs ElevenLabs v4 Turbo) and the video model (HeyGen Avatar IV vs Seedance 2.0 Mini). A paid probe (cap $3) decides both, including which voice models refuse intimate lines.

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
