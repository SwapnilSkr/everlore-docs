# NSFW Narration Models

How Everlore routes explicit turns, the candidate models we can use **on OpenRouter** as
alternatives/replacements for the current model, and how to test & switch them.

> All slugs below were verified against the OpenRouter models API and are live. Pricing is
> per **million** tokens (in / out) and context is the max window. Prices drift — re-check
> `https://openrouter.ai/api/v1/models` before committing.

---

## How routing works (recap)

Narration uses **two** models, chosen per turn (see `worker/processors/generation.processor.ts`):

- **SFW** → `AI_MODELS.narrationSfw` (currently `deepseek/deepseek-v3.2`)
- **NSFW** → `AI_MODELS.narrationNsfw` (currently `gryphe/mythomax-l2-13b`)

A turn routes to the NSFW model only when the world is mature-capable **and** the player
opted in **and** the scene classifier (`worker/lib/nsfw-classifier.ts`, backed by the
`nsfw_lexicon` collection) marks it explicit. Everything else stays on the SFW model, so the
NSFW model only has to be good at **one job: explicit prose**.

Switching models is centralized — change `narrationNsfw` in `src/ai/models.ts`, or set the
`NARRATION_NSFW_MODEL` env var. No other code changes needed.

---

## Current baseline


| Model       | Slug                     | Size | Context   | $/M (in/out) | Notes                                                                                                                             |
| ----------- | ------------------------ | ---- | --------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| MythoMax L2 | `gryphe/mythomax-l2-13b` | 13B  | **4K** ⚠️ | 0.06 / 0.06  | Cheap, classic uncensored RP. **4K context is the real problem** — too small for codex + memories + history; older prose quality. |


The 4K window is the main reason to move: our prompt (seed + lore + codex + memories +
recent turns) routinely wants more headroom than MythoMax allows.

---

## Candidate replacements (all live on OpenRouter)

> **On latency & speed:** uncensored finetunes mostly run on smaller/independent providers, so
> they're generally slower than the big SFW flash models — and the genuinely *fast + large
> context* NSFW options are limited. The standouts are **`cydonia`** (131K, ~440ms TTFT
> measured) and **`euryale`/`hermes`** (131K). TTFT/throughput aren't in OpenRouter's API, so
> the test script measures them locally (single samples; vary by route/region). Append
> **`:nitro`** to any slug to force-route to the fastest available provider.
>
> **Fast + large-context picks:** `cydonia` (best balance), `hermes`/`hermes3` (131K, cheap,
> steerable), `euryale` (131K, top quality). Small & fastest: `lunaris` (~0.9–1.1s TTFT, but
> only 8K context).

### Tier 1 — Cheap drop-in upgrades (small RP finetunes)


| Alias       | Slug                             | Size | Context | $/M (in/out) | Notes                                                                                                                                                                             |
| ----------- | -------------------------------- | ---- | ------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lunaris`   | `sao10k/l3-lunaris-8b`           | 8B   | 8K      | 0.04 / 0.05  | **Cheapest sensible upgrade & fast (~0.9–1.1s TTFT measured).** Sao10K's well-liked L3 RP blend; better coherence than MythoMax. Caveat: only 8K context.                                                                   |
| `rocinante` | `thedrummer/rocinante-12b`       | 12B  | **32K** | 0.17 / 0.43  | TheDrummer adventure/RP finetune; bold, uncensored, 8× the context.                                                                                                               |
| `unslop`    | `thedrummer/unslopnemo-12b`      | 12B  | 32K     | 0.40 / 0.40  | Nemo finetune tuned to avoid clichéd "slop" phrasing; fresher prose.                                                                                                              |
| `aionrp`    | `aion-labs/aion-rp-llama-3.1-8b` | 8B   | **32K** | 0.80 / 1.60  | **Ranks #1 on RPBench-Auto's character eval.** Purpose-built RP finetune — punches above 8B on in-character consistency. Priciest 8B here, but the most "RP-native" small option. |


### Tier 2 — Balanced (better prose, 24–36B)


| Alias     | Slug                                                            | Size | Context  | $/M (in/out) | Notes                                                                                                                                                                                     |
| --------- | --------------------------------------------------------------- | ---- | -------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cydonia` | `thedrummer/cydonia-24b-v4.1`                                   | 24B  | **131K** | 0.30 / 0.50  | **Recommended default — and the fastest large-context NSFW option (~440ms TTFT measured).** Uncensored creative-writing finetune with strong recall + prompt adherence; 131K context fits our full prompt comfortably.                                       |
| `skyfall` | `thedrummer/skyfall-36b-v2`                                     | 36B  | 32K      | 0.55 / 0.80  | Larger, richer prose; still affordable.                                                                                                                                                   |
| `venice`  | `cognitivecomputations/dolphin-mistral-24b-venice-edition:free` | 24B  | 32K      | **free**     | Uncensored Dolphin/Venice edition, **$0**. Great for testing; free tier = rate limits + variable availability, so not for production load.                                                |
| `minimax` | `minimax/minimax-m2-her`                                        | —    | **65K**  | 0.30 / 1.20  | Dialogue-first model **built specifically for immersive roleplay & character-driven chat**; strong persona/tone consistency over long multi-turn. Pricey output, but very "in-character." |
| `weaver`  | `mancer/weaver`                                                 | —    | 8K       | 0.75 / 1.00  | Classic Mancer RP host that recreates Claude-style verbosity. Uncensored, popular for narrative; smaller context + older coherence.                                                       |


### Tier 3 — High quality (70B+)


| Alias       | Slug                                   | Size     | Context  | $/M (in/out) | Notes                                                                                                                                                        |
| ----------- | -------------------------------------- | -------- | -------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `euryale`   | `sao10k/l3.3-euryale-70b`              | 70B      | **131K** | 0.65 / 0.75  | **Best quality/price for serious RP.** One of the most acclaimed NSFW RP finetunes; large context, still cheap for a 70B.                                    |
| `euryale31` | `sao10k/l3.1-euryale-70b`              | 70B      | **131K** | 0.85 / 0.85  | Prior-gen Euryale (L3.1 v2.2). Same creative-RP focus; keep as a fallback if 3.3 availability/price shifts.                                                  |
| `magnum`    | `anthracite-org/magnum-v4-72b`         | 72B      | 32K      | 3.00 / 5.00  | Tuned to emulate Claude-style prose; premium price — reserve for a paid tier.                                                                                |
| `hermes`    | `nousresearch/hermes-4-70b`            | 70B      | 131K     | 0.13 / 0.40  | Not an RP finetune but highly **steerable & near-unfiltered**; follows explicit system prompts. Cheap 70B; good neutral option.                              |
| `hermes3`   | `nousresearch/hermes-3-llama-3.1-70b`  | 70B      | **131K** | 0.30 / 0.30  | Prior Hermes gen — same steerable, near-unfiltered behavior; flat cheap pricing. Solid budget alternative to `hermes`.                                       |
| `hermes405` | `nousresearch/hermes-3-llama-3.1-405b` | **405B** | 131K     | 1.00 / 1.00  | Flagship steerable generalist; biggest model here, follows explicit system prompts well. Has a `**:free`** variant for $0 testing. Reserve for premium tier. |


---

## Recommendation

- **Move off MythoMax primarily for the context window.** Even the cheapest alternatives
give 8K–131K vs 4K.
- **Default pick: `cydonia` (`thedrummer/cydonia-24b-v4.1`)** — best balance of explicit
quality, 131K context, and ~$0.30–0.50/M.
- **Premium/quality tier: `euryale` (`sao10k/l3.3-euryale-70b`)** — noticeably stronger prose
for a small price bump; ideal for a paid NSFW tier.
- **Budget floor: `lunaris`** — if cost is the only concern.
- **Most "in-character" small model: `aionrp`** — RP-benchmark topper if persona consistency matters more than prose richness.
- **Free testing: `venice`** (or `hermes405:free`) — validate behavior at $0 before committing spend.
- **Fastest + large context: `cydonia`** (~440ms TTFT, 131K) — best speed/headroom combo; add `:nitro` to push for the fastest provider route.

Pick by A/B testing with the script below on real prompts — model "feel" matters more than
benchmarks for this use case.

---

## Testing

Aliases live in `scripts/test-nsfw-model.ts` (the `MODELS` map). **Every run tests BOTH an
explicit NSFW turn AND an ordinary SFW narration turn** (shared prompts in
`scripts/model-test-lib.ts`) so you can see each finetune's full boundary: is it *actually*
uncensored, and is it also a competent plain narrator? The harness streams the response and
reports **TTFT (time to first token), total time, approx tokens/sec**, and a refusal flag.

```bash
bun run scripts/test-nsfw-model.ts cydonia            # one model, both scenarios
bun run scripts/test-nsfw-model.ts euryale cydonia    # compare a few
bun run scripts/test-nsfw-model.ts all                # every alias
bun run scripts/test-nsfw-model.ts cydonia --nsfw     # NSFW scenario only
bun run scripts/test-nsfw-model.ts cydonia --sfw      # SFW scenario only
npm run test:nsfw                                     # default (mythomax baseline)
```

Compare prose quality, **TTFT/throughput**, and whether a model refuses or softens — side by side.

## Switching the production model

```bash
# .env
NARRATION_NSFW_MODEL=thedrummer/cydonia-24b-v4.1
```

…or edit `narrationNsfw` in `src/ai/models.ts`. Restart the worker to apply.