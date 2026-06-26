# World Creation Playbook (agent-driven, image-on)

**What this is:** the exact, repeatable procedure for an agent to create **real, publishable** worlds — GM / sentient / character — as the dev user, WITH a generated cover image, and leave them **published and ready to play**. The user then opens the app and jumps in. The agent does NOT auto-play and does NOT delete.

**Audience:** an autonomous agent with Bash + `jq` against the running local stack (`localhost:3000`).

**Sibling doc:** `AUTOCHAT_PLAYBOOK.md` covers logging in and *auto-playing* worlds. This doc is the *creation* half and **intentionally inverts its image policy** — see below.

---

## ⛔ Lifecycle policy for THIS playbook (read first)

These are **publish-ready / real** worlds the user will actually play. So, unlike the autochat QA loop:

| | This playbook |
|---|---|
| Cover image? | **YES — generate one** (§4). It costs real money; that's intended here. |
| Persist? | **YES — never delete.** |
| Auto-play? | **NO.** Stop at "published" (+ optional starter instance). The user plays. |

Record every `template_id` (and `instance_id` if you make one) in a handoff note so the user can find them. **When in doubt, keep the world.**

---

## 0. Preconditions

1. **Server + worker up** on `localhost:3000`:
   ```bash
   cd everlore-server
   bun run src/index.ts        # API           -> /tmp/everlore-api.log
   bun run worker/index.ts     # BullMQ worker  -> /tmp/everlore-worker.log
   ```
   (Worker isn't needed to *create* a world, but is needed the moment the user plays a turn.)
2. **Mongo + Redis** reachable (`.env` already points at dev).
3. **Auth is mocked.** Single dev user — phone `+919474877175`, OTP `123456` (any code works while Twilio env is empty), user id `6a210ba38e6db660dc8ef6a3`, tier `creator` (image gen requires `creator`/`premium` ✓), `nsfw_enabled:true`.

---

## 1. Log in (get a JWT)

```bash
BASE=http://localhost:3000
PHONE="+919474877175"
curl -sX POST "$BASE/auth/otp/send" -H 'content-type: application/json' -d "{\"phone\":\"$PHONE\"}" >/dev/null
TOKEN=$(curl -sX POST "$BASE/auth/otp/verify" -H 'content-type: application/json' \
  -d "{\"phone\":\"$PHONE\",\"code\":\"123456\"}" | jq -r '.token')
echo "$TOKEN"
```
`TOKEN` is a no-expiry dev bearer JWT.

**Check your create budget BEFORE spending on images** (`template_create` is capped 5 per rolling 24h):
```bash
curl -s "$BASE/templates/mine/quota" -H "authorization: Bearer $TOKEN"   # {can_create, remaining, retry_after}
```
To reset during a run: `redis-cli del rl:template_create:6a210ba38e6db660dc8ef6a3` (note it in handoff).

---

## 2. Pick the archetype

| Archetype | `kind` | `is_sentient` | Player is… | `protagonist`? | Stats |
|---|---|---|---|---|---|
| **GM world** | `world` | `false` | a protagonist inside a GM-narrated world | recommended | **≥1 required** |
| **Sentient world** | `world` | `true` | a persona talking *to* the world-as-a-character | no | **≥1 required** (kind=world rule) |
| **Character** | `character` | `true` | chatting 1:1 with one companion | no | optional (may be 0) |

Set `is_nsfw_capable:true` if the world should allow explicit content (the dev user is opted in).

---

## 3. Draft the content — two paths

### Path A — Autofill (recommended; gives you a coherent draft AND a ready image prompt)
One call drafts the whole thing from a short brief, and returns a **pre-decorated `image_prompt`** you can feed straight to image gen.
```bash
# target: "world" (GM or sentient) or "character"
DRAFT=$(curl -sX POST "$BASE/templates/autofill" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' -d '{
    "target":"world",
    "brief":"a rain-soaked cyber-noir megacity ruled by corp syndicates",
    "is_sentient":false,
    "is_nsfw_capable":true,
    "narrative_style":"cyberpunk"
  }')
echo "$DRAFT" | jq '.draft | {title, narrative_style, image_prompt}'
```
World draft returns: `title, description, seed_prompt, global_lore, narrative_style, style_notes, opening_line, scene_tags, stats, flags, image_prompt`.
Character draft returns: `name, tagline, persona, greeting, backstory, narrative_style, style_notes, image_prompt`.

Pull the ready image prompt: `IMGP=$(echo "$DRAFT" | jq -r '.draft.image_prompt')`.

### Path B — Manual
Write the fields yourself and build the decorated image prompt by hand (§4). Use this when you want exact control. Caps (`FIELD_LIMITS`): title ≤80, description ≤600, seed_prompt 10–2500, global_lore ≤3000, opening_line ≤600, image_prompt ≤1200, protagonist name ≤80 / text ≤400.

---

## 4. Generate the cover image

**The `/templates/image/generate` endpoint sends your prompt RAW — it does NOT decorate it.** So either use the autofill `image_prompt` (already decorated), or decorate a manual prompt yourself in this exact shape (mirrors `decorateImagePrompt`):

```
<concrete visual core, ≤400 chars>. <STYLE HINT for your narrative_style>. vertical portrait composition suited to a phone background, single clear focal subject, atmospheric depth, no text, no watermark, no logo, no UI.
```

**STYLE HINT** is chosen by `narrative_style` (one portrait image serves as BOTH the listing avatar AND the in-chat background, so always vertical/portrait). Pick the `narrative_style` that matches the world and the matching hint auto-applies. Available keys:
`tsundere, romcom, flirty, noir, slice_of_life, whimsical, epic_fantasy, grimdark, modern_casual, yandere, dark_romance, shonen, cyberpunk, kdrama, cozy_comfort, dark_academia, regency, horror, litrpg, chaotic_comedy` (anything else → a generic "high-quality character illustration, cinematic lighting").

Generate and read **`.url`** (NOT `.image_url` — that key is null and is the classic mistake that leaves a world with no cover):
```bash
IMG=$(curl -sX POST "$BASE/templates/image/generate" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' -d "{\"prompt\":$(jq -Rn --arg p "$IMGP" '$p')}" | jq -r '.url')
echo "cover=$IMG"
```
- Tier-gated (creator/premium ✓) and rate-limited (`image_generate`). On 429, wait and retry.
- Image gen runs BEFORE template validation, so only generate once the body is otherwise ready.
- For a **character** world, this same portrait is the character's avatar — make the visual core describe the *person*, not a landscape.

---

## 5. Create the template

Pass the generated cover as `image_url`. Pick the body for your archetype.

**GM world** (kind=world, is_sentient=false, needs a stat + protagonist):
```bash
TPL=$(curl -sX POST "$BASE/templates" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' -d '{
    "title":"Neon Divide","description":"A cyber-noir city where every favor has a price.",
    "kind":"world","is_sentient":false,"is_nsfw_capable":true,
    "seed_prompt":"You narrate a rain-soaked megacity ruled by corp syndicates. Grounded, modern, tense. Second person, present tense.",
    "global_lore":"Currency is creds. The Divide splits analog old-town from chrome uptown.",
    "narrative_style":"cyberpunk",
    "image_url":"'"$IMG"'",
    "protagonist":{"name":"Kade","persona":"a burned-out fixer","appearance":"worn coat, augmented left eye"},
    "base_stats_template":{"heat":{"default":0,"min":0,"max":100,"description":"how wanted you are"}},
    "opening_line":"The rain hasn'\''t stopped in three days. Your terminal blinks: one new job."
  }' | jq -r '._id'); echo "template=$TPL"
```

**Sentient world** — same as GM but `"is_sentient":true`, **omit `protagonist`**, keep ≥1 stat. The seed_prompt should be written as the world speaking/aware ("You are the city itself, addressing the traveler…").

**Character** — `"kind":"character","is_sentient":true`, no protagonist, stats optional (you may omit `base_stats_template`). Put the character's voice/persona in `seed_prompt` and a strong `opening_line` (their greeting). Use the autofill character draft's `persona`/`greeting`/`backstory` to fill these.

> Mapping autofill→create: world draft fields map almost 1:1 (`stats`→`base_stats_template`). Character draft `greeting`→`opening_line`, `persona`/`backstory`→fold into `seed_prompt`/`global_lore`.

---

## 6. Publish (this is what makes it playable)

```bash
curl -sX POST "$BASE/templates/$TPL/publish" -H "authorization: Bearer $TOKEN" >/dev/null
```
After publish, the world shows up for the user to open and start. **This is the normal stopping point** — the user starts their own instance in-app and plays.

**Optional — pre-create a starter instance** (only if the user wants a save already waiting):
```bash
INST=$(curl -sX POST "$BASE/instances" -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' -d "{\"template_id\":\"$TPL\"}" | jq -r '._id'); echo "instance=$INST"
```
(A creator can also start an instance on their OWN *unpublished* template for playtesting; non-owners get `403` until it's published.)

---

## 7. Verify + hand off

```bash
curl -s "$BASE/templates/mine/list" -H "authorization: Bearer $TOKEN" | jq '.[] | {_id, title, kind, is_sentient, is_published, image_url}'
```
Confirm each new world has `is_published:true` and a non-null `image_url`. Then write a short handoff listing every `template_id` (+ archetype, + `instance_id` if created) so the user knows exactly what to open.

---

## Gotchas (all real, all from the code)
- **`.url` not `.image_url`** on the image response. The #1 mistake.
- **The image endpoint does not decorate** — feed it the autofill `image_prompt`, or hand-build the decorated shape in §4. A bare core line gives a flat, off-style image.
- **kind=world needs ≥1 stat**; character may have zero.
- **`template_create` 5/24h** — check `/templates/mine/quota` before the image spend; the budget is consumed only on a *successful* create (a validation failure no longer burns a slot).
- **Image gen happens before validation** — have the body ready first so you don't pay for an image the create then rejects.
- **Stop at publish.** Do not auto-play and do not delete — these are the user's keepers.
