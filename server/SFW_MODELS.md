# SFW Narration Models

The model that handles **non-explicit narration** (the default path for almost every
turn), the candidate models we can use **on OpenRouter** as alternatives/replacements, and
how to test & switch them.

> All slugs below were verified against the OpenRouter models API and are live. Pricing is
> per **million** tokens (in / out) and context is the max window. Prices/availability drift —
> re-check `https://openrouter.ai/api/v1/models` before committing.

This is the **quality** path: it must be a great general storyteller — vivid prose, consistent
character voice, strong instruction-following (POV, tone, the italics/quotes format), and no
JSON/meta leakage. It does **not** need to be uncensored; explicit turns route to the NSFW
model instead (see [NSFW_MODELS.md](NSFW_MODELS.md)).

---

## How routing works (recap)

Narration uses **two** models, chosen per turn (`worker/processors/generation.processor.ts`):

- **SFW** → `AI_MODELS.narrationSfw` (currently `deepseek/deepseek-v3.2`) — this document
- **NSFW** → `AI_MODELS.narrationNsfw` — the explicit path

Switching is centralized — change `narrationSfw` in `src/ai/models.ts`, or set the
`NARRATION_SFW_MODEL` env var. No other code changes needed.

---

## Current baseline

| Model | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| DeepSeek V3.2 | `deepseek/deepseek-v3.2` | 131K | 0.25 / 0.38 | Current default. Strong, cheap, coherent prose with good instruction-following and large context. Solid all-rounder — the bar to beat. |

---

> **On latency:** OpenRouter's models API does not expose live TTFT/throughput, so the
> numbers below are **measured locally** by the test script (single samples — they vary by
> region, time of day, and which provider OpenRouter routes to). Treat them as ballpark.
> Append **`:nitro`** to any slug to force-route to the fastest provider.

## Candidate replacements (all live on OpenRouter)

### ⚡ Fast + HUGE context (1M tokens) — best latency-per-quality

| Alias | Slug | Ctx | $/M (in/out) | Speed (measured) | Notes |
|---|---|---|---|---|---|
| `geminiflashlite` | `google/gemini-2.5-flash-lite` | **1M** | 0.10 / 0.40 | **~950ms TTFT, 150–200 tok/s** | **Fastest strong narrator measured.** Huge context, very cheap, excellent throughput. Top speed pick. |
| `geminiflash` | `google/gemini-2.5-flash` | **1M** | 0.30 / 2.50 | ~0.7–1.5s TTFT, ~120 tok/s | Richer prose than lite, still very fast; 1M context. Strong default upgrade. |
| `deepseekflash` | `deepseek/deepseek-v4-flash` | **1M** | 0.10 / 0.20 | ~1.5–2s TTFT, ~25–100 tok/s | MoE built for fast/high-throughput inference; 1M context, baseline-cheap. Throughput varies by route. |
| `gemini3flash` | `google/gemini-3-flash-preview` | **1M** | 0.50 / 3.00 | — | Newest Gemini flash; sharper writing, preview availability. |

### Fast & cheap (small / MoE — lowest latency)

| Alias | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| `gptoss` | `openai/gpt-oss-120b` | 131K | 0.04 / 0.18 | Fast MoE (often routed to Groq/Cerebras = blazing). ⚠️ **Refuses explicit content** — pure-SFW only. |
| `nemo` | `mistralai/mistral-nemo` | 131K | **0.02 / 0.03** | Tiny, very fast, dirt cheap; serviceable prose for a fallback/free tier. |
| `llama31_8b` | `meta-llama/llama-3.1-8b-instruct` | 131K | 0.02 / 0.05 | 8B — lowest latency class; fine for short, snappy turns. |
| `gemma3` | `google/gemma-3-27b-it` | 131K | 0.08 / 0.16 | Punches above its size on atmosphere & description; excellent cheap creative writer. |
| `qwen3` | `qwen/qwen3-235b-a22b-2507` | **262K** | **0.07 / 0.10** | **Absurd value.** Big MoE, capable prose, huge context, fast routes. |

### Balanced (richer prose)

| Alias | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| `llama33` | `meta-llama/llama-3.3-70b-instruct` | 131K | 0.10 / 0.32 | Reliable, warm narrator; great instruction-following. Has a `:free` variant. |
| `glm46` | `z-ai/glm-4.6` | **203K** | 0.43 / 1.74 | Very strong creative-writing/RP coherence. |
| `kimi` | `moonshotai/kimi-k2.5` | **262K** | 0.40 / 1.90 | Excellent long-form prose & character consistency over long contexts. |
| `mistral` | `mistralai/mistral-medium-3.1` | 131K | 0.40 / 2.00 | Clean, literate prose; dependable instruction adherence. |

### Premium (lowest TTFT / best prose)

| Alias | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| `haiku` | `anthropic/claude-haiku-4.5` | 200K | 1.00 / 5.00 | **Lowest, most stable TTFT measured industry-wide (~600ms)**; crisp prose. Premium-ish but fast. |
| `sonnet` | `anthropic/claude-sonnet-4.5` | **1M** | 3.00 / 15.00 | Gold-standard narrative prose & character nuance; paid tier only. |
| `geminipro` | `google/gemini-2.5-pro` | **1M** | 1.25 / 10.00 | High-end reasoning + prose; premium tier. |

---

## Recommendation

- **Keep `deepseek` (v3.2) as the sensible default** — cheap, good, 131K. Move only if A/B
  clearly favors a candidate.
- **Fastest upgrade with the biggest context: `geminiflashlite`** — ~150–200 tok/s, sub-1s
  TTFT, 1M tokens, $0.10/$0.40. Best speed-per-dollar measured here.
- **Best value: `qwen3`** — cheaper than baseline, 262K context.
- **Lowest latency feel: `haiku`** — most stable time-to-first-token (premium price).
- **Premium prose: `sonnet`**. **Budget floor: `nemo` / `llama31_8b`**.

Pick by A/B testing with the script below on real turns — narration "feel" (voice, POV/format
discipline, no JSON leakage) and **TTFT** (what the player actually feels) matter more than raw
benchmarks.

---

## Testing

Aliases live in `scripts/test-sfw-model.ts` (the `MODELS` map). **Every run tests BOTH an SFW
narration turn AND an explicit NSFW turn** (shared prompts in `scripts/model-test-lib.ts`) so
you can see each model's full boundary: great narration *and* whether it refuses/softens
explicit content. The harness streams the response and reports **TTFT, total time, approx
tokens/sec**, and a refusal flag.

```bash
bun run scripts/test-sfw-model.ts geminiflashlite          # one model, both scenarios
bun run scripts/test-sfw-model.ts deepseek geminiflash gptoss  # compare a few
bun run scripts/test-sfw-model.ts all                      # every alias
bun run scripts/test-sfw-model.ts deepseek --sfw           # SFW scenario only
bun run scripts/test-sfw-model.ts haiku --nsfw             # NSFW scenario only
npm run test:sfw                                           # default (deepseek baseline)
```

## Switching the production model

```bash
# .env
NARRATION_SFW_MODEL=qwen/qwen3-235b-a22b-2507
```

…or edit `narrationSfw` in `src/ai/models.ts`. Restart the worker to apply.
