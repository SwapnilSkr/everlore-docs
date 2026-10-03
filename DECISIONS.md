# Decisions

Settled product and engineering calls, so they are not re-asked. Each entry says what was decided, why, and what would reopen it. Newest first.

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
