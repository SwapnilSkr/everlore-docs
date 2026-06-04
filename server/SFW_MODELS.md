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
model instead (see [NSFW_MODELS.md](NSFW_MODELS.md)). For read-aloud, see [TTS_MODELS.md](TTS_MODELS.md).

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

> **OpenRouter RP leaderboard (June 2026):** Real usage for roleplay/creative writing on
> OpenRouter heavily favors **1M flash models** — [V4 Flash](https://openrouter.ai/deepseek/deepseek-v4-flash),
> [MiMo 2.5](https://openrouter.ai/xiaomi/mimo-v2.5), [Owl Alpha](https://openrouter.ai/openrouter/owl-alpha) (free),
> Sonnet/Opus 4.6–4.8, V4 Pro, V3.2, Gemini 3 Flash, and [Gemma 4 26B](https://openrouter.ai/google/gemma-4-26b-a4b-it).
> See the full [roleplay collection](https://openrouter.ai/collections/roleplay). Finetunes (Cydonia, Euryale) rank lower
> in volume but remain the go-to for **uncensored** turns — see [NSFW_MODELS.md](NSFW_MODELS.md).

### ⚡ Fast + HUGE context (1M tokens) — best latency-per-quality

| Alias | Slug | Ctx | $/M (in/out) | Speed (measured) | Notes |
|---|---|---|---|---|---|
| `geminiflashlite` | `google/gemini-2.5-flash-lite` | **1M** | 0.10 / 0.40 | **~950ms TTFT, 150–200 tok/s** | **Fastest strong narrator measured.** Huge context, very cheap, excellent throughput. Top speed pick. |
| `geminiflash` | `google/gemini-2.5-flash` | **1M** | 0.30 / 2.50 | ~0.7–1.5s TTFT, ~120 tok/s | Richer prose than lite, still very fast; 1M context. Strong default upgrade. |
| `deepseekflash` | `deepseek/deepseek-v4-flash` | **1M** | 0.10 / 0.20 | ~1.5–2s TTFT, ~25–100 tok/s | MoE built for fast/high-throughput inference; 1M context, baseline-cheap. Throughput varies by route. |
| `deepseekv4pro` | `deepseek/deepseek-v4-pro` | **1M** | 0.44 / 0.87 | — | Natural successor to v3.2 baseline — same family, 8× context; strong A/B vs `deepseek`. |
| `qwen35flash` | `qwen/qwen3.5-flash-02-23` | **1M** | 0.07 / 0.26 | — | Cheapest 1M-window option found; hybrid MoE flash — test vs `geminiflashlite`. |
| `mimo25` | `xiaomi/mimo-v2.5` | **1M** | 0.14 / 0.28 | — | 2026 value pick: 1M context, half the cost of v4-pro; long-session narrator. |
| `mimo25pro` | `xiaomi/mimo-v2.5-pro` | **1M** | 0.44 / 0.87 | — | Heavier long-horizon/agent model; premium SFW tier candidate. |
| `qwen35plus` | `qwen/qwen3.5-plus-20260420` | **1M** | 0.30 / 1.80 | — | Multimodal Plus flagship (text/image/video in); 1M context. |
| `qwen35plus15` | `qwen/qwen3.5-plus-02-15` | **1M** | 0.26 / 1.56 | — | Slightly cheaper Plus variant; same 1M VL family. |
| `qwen37max` | `qwen/qwen3.7-max` | **1M** | 1.25 / 3.75 | — | Qwen 3.7 flagship; agent/long-horizon focus. Premium probe. |
| `owlalpha` | `openrouter/owl-alpha` | **1M** | **free** | — | **#3 on OR RP leaderboard.** Agentic foundation model; $0 (provider may log prompts). Great $0 A/B. |
| `gemini35flash` | `google/gemini-3.5-flash` | **1M** | 1.50 / 9.00 | — | Newest Gemini flash line; near-Pro coding/reasoning at flash latency. |
| `gemini3flash` | `google/gemini-3-flash-preview` | **1M** | 0.50 / 3.00 | — | Top-10 OR RP usage; strong agentic/chat. Preview availability. |
| `geminiflashliteprev` | `google/gemini-2.5-flash-lite-preview-09-2025` | **1M** | 0.10 / 0.40 | — | Alternate flash-lite snapshot; same price class as `geminiflashlite`. |

### Fast & cheap (small / MoE — lowest latency)

| Alias | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| `gptoss` | `openai/gpt-oss-120b` | 131K | 0.04 / 0.18 | Fast MoE (often routed to Groq/Cerebras = blazing). ⚠️ **Refuses explicit content** — pure-SFW only. |
| `nemo` | `mistralai/mistral-nemo` | 131K | **0.02 / 0.03** | Tiny, very fast, dirt cheap; serviceable prose for a fallback/free tier. |
| `llama31_8b` | `meta-llama/llama-3.1-8b-instruct` | 131K | 0.02 / 0.05 | 8B — lowest latency class; fine for short, snappy turns. |
| `gemma3` | `google/gemma-3-27b-it` | 131K | 0.08 / 0.16 | Punches above its size on atmosphere & description; excellent cheap creative writer. |
| `qwen3` | `qwen/qwen3-235b-a22b-2507` | **262K** | **0.07 / 0.10** | **Absurd value.** Big MoE, capable prose, huge context, fast routes. |
| `seed16flash` | `bytedance-seed/seed-1.6-flash` | **262K** | 0.08 / 0.30 | Ultra-fast 256K; latency experiments. Has `:free` variants for $0 probing. |
| `qwennext80` | `qwen/qwen3-next-80b-a3b-instruct` | **262K** | 0.09 / 1.10 | Non-thinking instruct MoE; cheap. Has `:free` variant. |
| `gemma4` | `google/gemma-4-26b-a4b-it` | **262K** | 0.06 / 0.33 | **#9 on OR RP leaderboard.** MoE Gemma 4; top RP usage among small models. Has `:free`. |
| `gemma431` | `google/gemma-4-31b-it` | **262K** | 0.12 / 0.37 | Dense Gemma 4 multimodal; richer than 26B MoE. |
| `mimoflash` | `xiaomi/mimo-v2-flash` | **262K** | 0.10 / 0.30 | Open-weight MiMo flash; cheaper than 2.5, still strong. |
| `ling26flash` | `inclusionai/ling-2.6-flash` | **262K** | **0.01 / 0.03** | Absurdly cheap 104B MoE (7.4B active); budget floor beyond `nemo`. |
| `deepseekv31` | `deepseek/deepseek-chat-v3.1` | **164K** | 0.21 / 0.79 | Hybrid thinking/non-thinking V3.1; between v3.2 and V4. |
| `step35flash` | `stepfun/step-3.5-flash` | **262K** | 0.09 / 0.30 | StepFun open MoE flash; fast 256K generalist. |

### Balanced (richer prose)

| Alias | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| `llama33` | `meta-llama/llama-3.3-70b-instruct` | 131K | 0.10 / 0.32 | Reliable, warm narrator; great instruction-following. Has a `:free` variant. |
| `glm46` | `z-ai/glm-4.6` | **203K** | 0.43 / 1.74 | Very strong creative-writing/RP coherence. |
| `glm47` | `z-ai/glm-4.7` | **203K** | 0.40 / 1.75 | Upgrade from 4.6; stronger agent/reasoning + creative tasks. |
| `glm47flash` | `z-ai/glm-4.7-flash` | **203K** | 0.06 / 0.40 | Budget 30B-class GLM; fast creative tier. |
| `glm5` | `z-ai/glm-5` | **203K** | 0.60 / 1.92 | Latest Z.ai flagship; production-grade long-horizon work. |
| `kimi` | `moonshotai/kimi-k2.5` | **262K** | 0.40 / 1.90 | Excellent long-form prose & character consistency over long contexts. |
| `kimi26` | `moonshotai/kimi-k2.6` | **262K** | 0.68 / 3.42 | Successor to k2.5; multimodal + agents. Has `:free` variant. |
| `mistral` | `mistralai/mistral-medium-3.1` | 131K | 0.40 / 2.00 | Clean, literate prose; dependable instruction adherence. |
| `mistralsmall32` | `mistralai/mistral-small-3.2-24b-instruct` | 128K | 0.08 / 0.20 | **Base model behind Cydonia** — censored, but cheap SFW narration reference. |
| `qwen35_35b` | `qwen/qwen3.5-35b-a3b` | **262K** | 0.14 / 1.00 | Mid-tier Qwen 3.5 VL MoE. |
| `qwen35_122b` | `qwen/qwen3.5-122b-a10b` | **262K** | 0.26 / 2.08 | Larger Qwen 3.5 VL MoE. |
| `llama4maverick` | `meta-llama/llama-4-maverick` | **1M** | 0.15 / 0.60 | Llama 4 128E MoE multimodal; strong generalist. |
| `llama4scout` | `meta-llama/llama-4-scout` | **10M** | 0.08 / 0.30 | Extreme context (10M advertised); MoE — test codex+history headroom. |
| `grok43` | `x-ai/grok-4.3` | **1M** | 1.25 / 2.50 | xAI reasoning + agents; 1M ctx. |
| `grok420` | `x-ai/grok-4.20` | **2M** | 1.25 / 2.50 | Largest ctx in this list; agentic tool use. |
| `glm45air` | `z-ai/glm-4.5-air` | 131K | 0.13 / 0.85 | Lightweight GLM MoE agent model. Has `:free`. |

### Premium (lowest TTFT / best prose)

| Alias | Slug | Ctx | $/M (in/out) | Notes |
|---|---|---|---|---|
| `haiku` | `anthropic/claude-haiku-4.5` | 200K | 1.00 / 5.00 | **Lowest, most stable TTFT measured industry-wide (~600ms)**; crisp prose. Premium-ish but fast. |
| `sonnet` | `anthropic/claude-sonnet-4.5` | **1M** | 3.00 / 15.00 | Gold-standard narrative prose & character nuance; paid tier only. |
| `sonnet46` | `anthropic/claude-sonnet-4.6` | **1M** | 3.00 / 15.00 | Newer Sonnet with 1M ctx; frontier coding/agents + prose. |
| `opus46` | `anthropic/claude-opus-4.6` | **1M** | 5.00 / 25.00 | Top-tier prose & long workflows; highest cost. |
| `opus47` | `anthropic/claude-opus-4.7` | **1M** | 5.00 / 25.00 | **#5 on OR RP leaderboard.** Long-running async agents + knowledge work. |
| `opus48` | `anthropic/claude-opus-4.8` | **1M** | 5.00 / 25.00 | Latest Opus; top OR RP usage in premium tier. |
| `geminipro` | `google/gemini-2.5-pro` | **1M** | 1.25 / 10.00 | High-end reasoning + prose; premium tier. |
| `gpt51` | `openai/gpt-5.1` | 400K | 1.25 / 10.00 | Large ctx; GPT-4o lineage noted for natural creative writing on OR. |
| `gpt5chat` | `openai/gpt-5-chat` | 128K | 1.25 / 10.00 | Chat-optimized GPT-5; natural multimodal conversation. |
| `gpt52chat` | `openai/gpt-5.2-chat` | 128K | 1.75 / 14.00 | Fast “Instant” GPT-5.2 chat variant; low-latency dialogue. |
| `glm51` | `z-ai/glm-5.1` | **203K** | 0.98 / 3.08 | GLM 5.1; major coding/agent upgrade over GLM-5. |

### Free-tier probes (rate-limited, $0)

`openrouter/owl-alpha`, `google/gemma-4-26b-a4b-it:free`, `qwen/qwen3-next-80b-a3b-instruct:free`,
`moonshotai/kimi-k2.6:free`, `openai/gpt-oss-120b:free`, `meta-llama/llama-3.3-70b-instruct:free`,
`z-ai/glm-4.5-air:free` — useful for boundary testing before spend; not for production load.

---

## Recommendation

- **Keep `deepseek` (v3.2) as the sensible default** — cheap, good, 131K. Move only if A/B
  clearly favors a candidate.
- **Fastest upgrade with the biggest context: `geminiflashlite`** — ~150–200 tok/s, sub-1s
  TTFT, 1M tokens, $0.10/$0.40. Best speed-per-dollar measured here.
- **Baseline successor (1M): `deepseekv4pro`** — same DeepSeek family as v3.2, 8× context.
- **Cheapest 1M probe: `qwen35flash`** or **`mimo25`** — sub-$0.30/M output; A/B before switching prod.
- **Free RP leaderboard pick: `owlalpha`** — $0, 1M ctx, heavy real RP traffic on OR (verify prose/format).
- **OR RP usage dark horse: `gemma4`** — cheap 262K, surprisingly high RP volume for its size.
- **Extreme context experiment: `llama4scout`** — 10M window if codex+memories+history ever outgrow 1M.
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
bun run scripts/test-sfw-model.ts deepseekv4pro mimo25 qwen35flash  # new 1M candidates
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
