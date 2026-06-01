# Image Generation Models

The model that generates **world/character avatars + chat backgrounds** (one
portrait image per template, used for both), the OpenRouter candidates, and how
to switch.

> Verified against the OpenRouter image API. Generation uses the chat-completions
> endpoint with `modalities: ["image"]`; the bytes come back as a base64 data URL
> in `choices[0].message.images[0].image_url.url`. Use **`["image"]`** (not
> `["image","text"]`) — image-only models like Seedream reject the latter.

## How it works (recap)

- Model is `AI_MODELS.image` (`src/ai/models.ts`), env override **`IMAGE_MODEL`**.
- `src/ai/image.ts` → `generateImage(prompt)` calls OpenRouter, returns decoded bytes + mime.
- `src/services/image.service.ts` → `suggestPrompt()` (auto-suggest, voice-aware) + `generatePreview()`.
- `src/services/storage.service.ts` → uploads to a **private S3 bucket**; returns a **CloudFront** URL (bucket is locked to the distribution via OAC). Previews land under `previews/` (lifecycle-expired after 1 day); the chosen one is promoted to `media/` on save.

## Current baseline

| Model | Slug | Price | Notes |
|---|---|---|---|
| **Seedream 4.5** (ByteDance) | `bytedance-seed/seedream-4.5` | **$0.04 flat / image** | **Default.** Purpose-built for stylized/anime illustration; cheap, fast, predictable flat pricing. Image-only → `modalities:["image"]`. Returns JPEG. |

## Alternates (all live on OpenRouter)

| Alias | Slug | Price | When to use |
|---|---|---|---|
| Flux.2 Klein 4B | `black-forest-labs/flux.2-klein-4b` | ~$0.015/MP (cheapest/fastest) | Fast/cheap; stylized + cinematic. Strong budget alt. |
| Gemini 2.5 Flash Image (Nano Banana) | `google/gemini-2.5-flash-image` | $0.30/$2.50 per M tok | **Photoreal-leaning**, best face/character consistency. Use for realistic/grounded worlds. Accepts `["image"]` or `["image","text"]`. |
| Gemini 3.1 Flash Image | `google/gemini-3.1-flash-image-preview` | $0.50/$3 per M tok | Newer, Pro-level quality at Flash speed. |
| Premium tier | `google/gemini-3-pro-image-preview`, `openai/gpt-5-image`, `x-ai/grok-imagine-image-quality` | $$$ | Reserve for a paid tier. |

**Anime verdict:** Seedream 4.5 ≫ Nano Banana for stylized/anime (Nano Banana is a
photorealism model). Default stays Seedream; switch to Gemini per-world only if a
world wants photoreal art.

## Switching

```bash
# .env
IMAGE_MODEL=black-forest-labs/flux.2-klein-4b
```

…or edit `AI_MODELS.image` in `src/ai/models.ts`. No other code changes needed.

## AWS infra (provisioned)

- **Bucket** `everlore-media-209479287671` (ap-south-1) — all public access blocked, SSE-S3, `previews/` lifecycle expiry (1 day).
- **CloudFront** `d2e8gzt8asvxl4.cloudfront.net` — Origin Access Control (SigV4); bucket policy allows only this distribution to read.
- **IAM** `everlore-s3-uploader` — scoped `PutObject`/`GetObject`/`DeleteObject` on the bucket; keys in `.env` (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`).
- Env: `AWS_REGION`, `S3_BUCKET`, `CDN_BASE_URL`.
