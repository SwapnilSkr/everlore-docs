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

| Model | Slug | Size | Context | $/M (in/out) | Notes |
|---|---|---|---|---|---|
| MythoMax L2 | `gryphe/mythomax-l2-13b` | 13B | **4K** ⚠️ | 0.06 / 0.06 | Cheap, classic uncensored RP. **4K context is the real problem** — too small for codex + memories + history; older prose quality. |

The 4K window is the main reason to move: our prompt (seed + lore + codex + memories +
recent turns) routinely wants more headroom than MythoMax allows.

---

## Candidate replacements (all live on OpenRouter)

### Tier 1 — Cheap drop-in upgrades (small RP finetunes)

| Alias | Slug | Size | Context | $/M (in/out) | Notes |
|---|---|---|---|---|---|
| `lunaris` | `sao10k/l3-lunaris-8b` | 8B | 8K | 0.04 / 0.05 | **Cheapest sensible upgrade.** Sao10K's well-liked L3 RP blend; better coherence than MythoMax at similar cost. |
| `rocinante` | `thedrummer/rocinante-12b` | 12B | **32K** | 0.17 / 0.43 | TheDrummer adventure/RP finetune; bold, uncensored, 8× the context. |
| `unslop` | `thedrummer/unslopnemo-12b` | 12B | 32K | 0.40 / 0.40 | Nemo finetune tuned to avoid clichéd "slop" phrasing; fresher prose. |

### Tier 2 — Balanced (better prose, 24–36B)

| Alias | Slug | Size | Context | $/M (in/out) | Notes |
|---|---|---|---|---|---|
| `cydonia` | `thedrummer/cydonia-24b-v4.1` | 24B | **131K** | 0.30 / 0.50 | **Recommended default.** Uncensored creative-writing finetune with strong recall + prompt adherence; huge context fits our full prompt comfortably. |
| `skyfall` | `thedrummer/skyfall-36b-v2` | 36B | 32K | 0.55 / 0.80 | Larger, richer prose; still affordable. |
| `venice` | `cognitivecomputations/dolphin-mistral-24b-venice-edition:free` | 24B | 32K | **free** | Uncensored Dolphin/Venice edition, **$0**. Great for testing; free tier = rate limits + variable availability, so not for production load. |

### Tier 3 — High quality (70B+)

| Alias | Slug | Size | Context | $/M (in/out) | Notes |
|---|---|---|---|---|---|
| `euryale` | `sao10k/l3.3-euryale-70b` | 70B | **131K** | 0.65 / 0.75 | **Best quality/price for serious RP.** One of the most acclaimed NSFW RP finetunes; large context, still cheap for a 70B. |
| `magnum` | `anthracite-org/magnum-v4-72b` | 72B | 32K | 3.00 / 5.00 | Tuned to emulate Claude-style prose; premium price — reserve for a paid tier. |
| `hermes` | `nousresearch/hermes-4-70b` | 70B | 131K | 0.13 / 0.40 | Not an RP finetune but highly **steerable & near-unfiltered**; follows explicit system prompts. Cheap 70B; good neutral option. |

---

## Recommendation

- **Move off MythoMax primarily for the context window.** Even the cheapest alternatives
  give 8K–131K vs 4K.
- **Default pick: `cydonia` (`thedrummer/cydonia-24b-v4.1`)** — best balance of explicit
  quality, 131K context, and ~$0.30–0.50/M.
- **Premium/quality tier: `euryale` (`sao10k/l3.3-euryale-70b`)** — noticeably stronger prose
  for a small price bump; ideal for a paid NSFW tier.
- **Budget floor: `lunaris`** — if cost is the only concern.
- **Free testing: `venice`** — validate behavior at $0 before committing spend.

Pick by A/B testing with the script below on real prompts — model "feel" matters more than
benchmarks for this use case.

---

## Testing

Aliases live in `scripts/test-nsfw-model.ts` (the `MODELS` map). Run:

```bash
bun run scripts/test-nsfw-model.ts cydonia          # one model
bun run scripts/test-nsfw-model.ts euryale cydonia  # compare a few
bun run scripts/test-nsfw-model.ts all              # every alias
npm run test:nsfw                                   # default (mythomax baseline)
```

Each run prints the response + latency so you can compare prose quality and speed side by side.

## Switching the production model

```bash
# .env
NARRATION_NSFW_MODEL=thedrummer/cydonia-24b-v4.1
```

…or edit `narrationNsfw` in `src/ai/models.ts`. Restart the worker to apply.
