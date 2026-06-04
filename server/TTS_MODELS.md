# Text-to-Speech (TTS) Models

Models that **read narration aloud** (world/character voice-over, accessibility, optional
“listen to this turn”), the OpenRouter candidates, and how to test & switch them.

> Verified against OpenRouter's **`/api/v1/audio/speech`** endpoint and
> `GET https://openrouter.ai/api/v1/models?output_modalities=speech` (9 dedicated TTS
> models as of last check). Pricing is **per input character** unless noted. Re-check the
> [TTS collection](https://openrouter.ai/collections/text-to-speech-models) before committing.

Everlore narration still **generates prose via LLMs** ([SFW_MODELS.md](SFW_MODELS.md) /
[NSFW_MODELS.md](NSFW_MODELS.md)). TTS is a separate layer: turn text → audio bytes.

---

## How it works (recap)

- **API:** `POST https://openrouter.ai/api/v1/audio/speech` (OpenAI Audio Speech–compatible).
- **Code:** `src/ai/tts.ts` → `synthesizeSpeech({ model, input, voice, … })`.
- **Default slug:** `AI_MODELS.tts` in `src/ai/models.ts`, env override **`TTS_MODEL`**.
- **Not streamed in v1:** the endpoint returns a full audio bytestream (use `mp3` for
  playback files, `pcm` where required — see per-model notes).

### SFW vs NSFW for TTS

Unlike chat models, TTS rarely “refuses” in prose — it either **returns audio** or **HTTP
error** (policy / invalid voice / wrong `response_format`). The test script reads the same
**SFW narration line** and **NSFW explicit line** used conceptually in the narration A/B
scripts so you can compare:

- Does the provider block mature read-aloud?
- Does quality/emotion break on intimate wording?
- Latency and bytes for a ~2-sentence turn

---

## OpenRouter TTS leaderboard (June 2026)

Real usage on the [text-to-speech collection](https://openrouter.ai/collections/text-to-speech-models)
favors **`google/gemini-3.1-flash-tts-preview`**, then **`hexgrad/kokoro-82m`**, then
**`x-ai/grok-voice-tts-1.0`**. Finetunes (Orpheus, Zonos, CSM) have lower volume but can
sound more “acted” for fiction.

---

## Current baseline

| Model | Slug | Voice (test default) | Format | $/char (in) | Notes |
|---|---|---|---|---|---|
| Kokoro 82M | `hexgrad/kokoro-82m` | `af_bella` | mp3 | ~$0.62/M | **Default.** Cheapest on OR; 8 languages, 54 preset voices; great for cost floor. |

---

## Dedicated TTS models (`output_modalities=speech`)

All use **`/api/v1/audio/speech`**. Voices vary by provider — confirm on each model’s
OpenRouter page before production.

### Tier 1 — Best balance (quality × cost × latency)

| Alias | Slug | Voice (test) | Format | $/char (in) | Notes |
|---|---|---|---|---|---|
| `geminitts` | `google/gemini-3.1-flash-tts-preview` | `Zephyr` | **pcm** | $1/M in, $20/M out | **Top OR usage.** 70+ langs; 200+ inline tags (`[whispers]`, `[laughs]`); 2-speaker scenes; **must use `pcm`** on OR (mp3 → 400). |
| `groktts` | `x-ai/grok-voice-tts-1.0` | `Eve` | mp3 | $15/M | 20+ langs; tags for pause/emphasis; voices Eve, Ara, Rex, Sal, Leo; 15K chars/req. |
| `maivoice` | `microsoft/mai-voice-2` | `en-US-Harper:MAI-Voice-2` | mp3 | $22/M | Azure-backed; SSML styles via `provider.options.azure` (cheerful, sad, …); speed 0.5–2×. |
| `kokoro` | `hexgrad/kokoro-82m` | `af_bella` | mp3 | $0.62/M | **Cheapest.** Open-weight; EN/ES/FR/HI/IT/JA/PT/ZH. |

### Tier 2 — Narration / dialogue character

| Alias | Slug | Voice (test) | Format | $/char (in) | Notes |
|---|---|---|---|---|---|
| `orpheus` | `canopylabs/orpheus-3b-0.1-ft` | `tara` | mp3 | $7/M | English; 7 preset voices; strong prosody for fiction narration. |
| `voxtral` | `mistralai/voxtral-mini-tts-2603` | `alloy` | mp3 | $16/M | Zero-shot voice cloning + multilingual; verify voice id on model page. |
| `csm` | `sesame/csm-1b` | `alloy` | mp3 | $7/M | Conversational / read-speech styles; dialogue-oriented. |
| `zonosh` | `zyphra/zonos-v0.1-hybrid` | `alloy` | mp3 | $7/M | EN US/UK accents; hybrid arch. |
| `zonost` | `zyphra/zonos-v0.1-transformer` | `alloy` | mp3 | $7/M | Same voice coverage as hybrid; transformer-only stack. |

---

## Multimodal audio-out (chat API — different path)

These appear in the general models list with **`text` + `audio` output** but are **not**
in the `output_modalities=speech` filter. They use **chat completions** (modalities /
audio parts), not `/audio/speech`:

| Slug | $/M (in/out) | Notes |
|---|---|---|
| `openai/gpt-audio-mini` | 0.60 / 2.40 | Cost-efficient speech+text; upgraded decoder. |
| `openai/gpt-audio` | 2.50 / 10.00 | Full GPT Audio; natural voices. |

Use when you need **dialogue + reasoning in one call**; for “read this turn text” prefer
dedicated TTS above. Not wired in `test-tts-model.ts` yet.

---

## Not on OpenRouter API (docs only)

| Slug | Notes |
|---|---|
| `openai/gpt-4o-mini-tts-2025-12-15` | Listed in [OR TTS docs](https://openrouter.ai/docs/guides/overview/multimodal/tts) but **not** in `?output_modalities=speech` — re-check before use. |
| `arliai/qwq-32b-arliai-rpr-v1` | RP-tuned QwQ; was on OR web, absent from speech API. |
| `nothingiisreal/mn-celeste-12b` | Nemo Celeste RP/story; absent from speech API. |

**Music (not narration TTS):** `google/lyria-3-*` — song generation, not read-aloud.

**Speech-in only:** `mistralai/voxtral-small-24b-2507` — audio **input**, text out (STT-ish).

---

## Recommendation

- **Default: `kokoro`** — cheapest, mp3 out, fast smoke tests; good budget “read turn” tier.
- **Best quality / tags / multilingual: `geminitts`** — use **`pcm`** + convert for mobile if needed.
- **Expressive fiction + inline performance tags: `groktts`** or **`geminitts`**.
- **Azure enterprise voices: `maivoice`** with SSML `style` for scene tone.
- **NSFW boundary:** run `test-tts-model.ts` with `--nsfw` on finalists — providers differ more than LLM finetunes.

Pick by listening with `--save` on real turn excerpts, not benchmarks.

---

## Testing

Aliases live in `scripts/test-tts-model.ts`. **Every run synthesizes BOTH an SFW line and
an NSFW line** (`scripts/tts-test-lib.ts`). Reports latency, byte size, content-type, and
policy-style failures.

```bash
bun run scripts/test-tts-model.ts kokoro                 # default baseline
bun run scripts/test-tts-model.ts geminitts groktts      # compare top OR usage
bun run scripts/test-tts-model.ts all
bun run scripts/test-tts-model.ts kokoro --save            # scripts/tts-output/*.mp3|.pcm
bun run scripts/test-tts-model.ts geminitts --nsfw         # explicit line only
npm run test:tts                                           # default (kokoro)
```

Requires `OPENROUTER_API_KEY` in `.env`.

---

## Switching the production model

```bash
# .env
TTS_MODEL=google/gemini-3.1-flash-tts-preview
```

…or edit `tts` in `src/ai/models.ts`. Pair with a voice id your client sends (or map
per-character voice in template metadata). Restart services after change.

### Example call

```typescript
import { synthesizeSpeech } from '../src/ai/tts'

const { data, contentType } = await synthesizeSpeech({
  model: 'hexgrad/kokoro-82m',
  input: turnProse,
  voice: 'af_bella',
  responseFormat: 'mp3',
})
```

---

## Related docs

- [SFW_MODELS.md](SFW_MODELS.md) — prose generation (non-explicit)
- [NSFW_MODELS.md](NSFW_MODELS.md) — explicit prose generation
- [IMAGE_MODELS.md](IMAGE_MODELS.md) — avatar/background images (same OpenRouter pattern)
