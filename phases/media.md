# Phase: voiced lines, scene pictures and talking-character video

Status: **contract, Oct 16 2026.** All three models are chosen (§9.1, owner after the probe). Probe: `.agents/reports/media-probe.md`.
Decisions behind this: `DECISIONS.md` → "Voiced lines, scene images and talking-character video in chat worlds".

Chat worlds only. The interactive (walkable) world is out of scope.

---

## 1. Principles

1. **The turn is untouched.** Nothing in this phase runs before or during the narrator stream, so TTFT is unchanged. The work added to the post-stream step is model-free.
2. **Nothing is generated without a tap.** No voice, portrait, picture or video exists until a player asks for one. Once made, it is cached and reused.
3. **No clutter.** The chat gets **no new always-visible buttons**. Spoken lines are tapped directly; pictures live in the turn's existing long-press menu. A finished picture or video appears once, compact, under its turn.
4. **Pictures and video may be suggestive, never explicit, for everyone** (owner, Oct 16 2026). This is checked by code before and after generation (§7). Voice follows the player's mature opt-in.
5. **Only faces we painted.** No uploaded image is ever animated or used as a face reference.
6. **A quiet failure, never a broken one.** Anything we can't voice or picture simply has no action, or ends in calm in-world copy. Never an error dialog.

---

## 2. Spoken lines

### 2.1 Detection, after the stream, model-free

At the end of the existing post-stream step, the streamed `ai_response` is scanned for quoted speech by **punctuation structure only**, with no model call and no word lists. The result is saved on the event as `spoken_lines` and published (§6).

A span is a **candidate line** only if every rule holds:

| Rule | Why |
|---|---|
| It opens and closes with a matching double-quote pair: `“…”`, or straight `"…"` paired in order | Single quotes collide with apostrophes (`don't`, `'tis`) |
| Single quotes `‘…’` / `'…'` count **only** when the turn has no double quotes, every single quote pairs, and the quote characters are not between two letters (between letters = apostrophe) | British-style dialogue still works; contractions never open a line |
| It contains no paragraph break | A missing closing quote must not swallow the next paragraph |
| It has at least 2 words and at most 400 characters | It drops one-word scare quotes and runaway spans |
| It is not part of the player's own echoed words | We never voice the player |

**Prose hygiene may not have run on this turn.** Its fixes apply to later prompts, never to an already-streamed turn, so detection assumes nothing about our conventions. Odd output degrades to fewer lines, never to a wrong one:

- **An unclosed quote** pairs with nothing, so it gets no line. The rest of the turn is unaffected.
- **Narration inside quotes** (`"I'm fine, she lied, really."`) passes detection. It is caught at the tap (§2.2): the voiced text is the spoken part only.
- **Speech without quotes** (em-dash dialogue, italic thoughts) is not detected and can't be voiced. That's accepted.
- **Nested quotes** (a quote within speech): only the outer span is a candidate.

```ts
spoken_lines: Array<{
  line_id: string        // stable within the event: "l0", "l1", …
  start: number          // UTF-16 offsets into ai_response, quotes included
  end: number
}>
```

Replays and edits produce a new or changed `ai_response`, so their lines are recomputed. Rewinds remove the event and its lines.

### 2.2 Voicing, on the first tap

The first time a player taps any line in a turn, one cheap structured call processes **all** candidate lines of that turn at once. For each line it returns:

```ts
{
  line_id: string
  is_speech: boolean            // false for a quoted sign, letter, title, or thought
  speaker_name: string | null   // how the prose refers to the speaker, as written: a name or a description ("the Thorne rider"), never an id
  spoken_text: string           // the words actually said, narration removed
  tags: string[]                // 0–1 from the allowed performed-tag list (§2.3)
}
```

The server accepts a line only if:
- `spoken_text`, ignoring whitespace, is a **verbatim substring** of that line's span. This is the evidence rule: the model can drop narration but can't invent words.
- the speaker isn't the player.
- `tags` are filtered to the allowed list, at most 1.

**Every other spoken line is voiced** (owner, Oct 16 2026). A line is `unvoiceable` only when it isn't speech (a sign, letter, title or thought), when it is the player's own words, when `spoken_text` fails the verbatim rule, or when it's a mature line without the opt-in (§2.3). A missing card, an unnamed speaker, an unclear gender or a child is never a reason to refuse.

**Media never creates a card.** Who gets a card is the story pipeline's decision (codex extraction, its recurrence rule for unnamed people, merges), unchanged. The speaker handle resolves through the codex name/alias resolution:
- **A card matches:** the line uses the card's voice (§2.3), and the card's portrait for pictures and video.
- **No card matches:** the line is voiced with a voice chosen from the line's voice traits, deterministically from `instance_id` + the normalized handle, so the same handle in a save keeps the same voice. With no handle at all, the event and line ids seed it. The choice is stored on the line (`spoken_lines[i].voice.voice_id`), never on a card. Pictures draw this person from the prose. **"See them say it" is not offered** for an uncarded speaker: there is no face to animate.
- **The speaker later earns a card:** from then on, the card's voice is used. Already-voiced lines keep the voice they were played with.
- **Merges carry media.** When the codex merges two cards, the survivor keeps its own voice, portrait and look, and takes the source's when it has none.

**Voice traits never refuse.** An unclear gender picks the best fit from the traits that are known. **A child is voiced with the lightest adult voice** (owner, Oct 16 2026): the catalogue data maps `age_band: child` to the youngest light adult voices. The catalogue has no child voices.

Any line that fails is marked `unvoiceable`: the tap gets a soft haptic and nothing else, and the line is never offered again. Results are saved on the event (`spoken_lines[i].voice`) so the call runs once per turn.

### 2.3 Voice

- **Model: `elevenlabs/eleven-v4-turbo`** via OpenRouter `/audio/speech` (env `MEDIA_VOICE_MODEL`). It returns mp3; no transcoding.
  - Chosen by the owner on Oct 16 2026 after listening to the probe. It had the best performance and the cleanest transcripts, and was the fastest (~0.8 s per line). It refused none of the test lines, intimate ones included.
  - Price: $20 per million input characters, tags included, ≈ $0.002 (₹0.17) per ~100-character line. Fish Drama 3 was 10× cheaper but less good, and its launch price may be a promotion.
- **No automatic fallback model.** Another model would give the character a different voice. A provider failure is `media_failed` and the slot is released.
- **The voice catalogue is data**, not code: a file listing each of the model's 21 preset voices (`george`, `sarah`, … as `supported_voices` on `/api/v1/models`) with authored traits (gender presentation, age band, timbre). Changing models means a new catalogue file, not new code.
- **Voice per character.** The first time a card speaks, a voice is chosen deterministically and saved as `voice: { model, voice_id }`.
  - The fit uses the gender, age and temperament known on the card (structural fields, never a word list). Ties are broken by catalogue order.
  - **Within a save, prefer a voice no other speaking card uses.** World cast voices count as used in every save of that world.
  - With 21 voices, a busy save can run out. Then the fitting voice whose current holder spoke least recently is reused. Two characters only share a voice when there is no free fitting one.
  - World cast voices are stored on the world's cast entry and shared by every save, the same as portraits.
- **Feeling** comes from the written line plus at most one performed tag, placed at the start of the line. **No style note** is sent: on OpenRouter it is read aloud (proven in Genlife 7.19).
- **Allowed tags (probe-proven): `[shouting]`, `[whispers]`, `[sighs]`, `[breathless]`.** No transcript showed any of them spoken aloud on either voice. Two tags together and any other tag are untested. Widen the list only after a new probe.
- **Mature lines** are voiced only for players with `preferences.nsfw_enabled`. For everyone else, a line from an event with `nsfw_intent: true` is `unvoiceable`.
- **Cache key:** `event_id + line_id + voice_id + model`. A replayed line costs nothing.
- **Cost per call** is computed from the characters actually sent × the catalogue rate, then replaced by the OpenRouter generation cost when the lookup succeeds. During the probe, OpenRouter's `/generation` lookup returned 404 for most speech calls.

---

## 3. Portraits

Rules as decided (`DECISIONS.md`). Storage:

```ts
portrait?: {
  asset_key: string                   // S3 key; URL from CDN_BASE_URL
  source: 'generated' | 'authored'    // 'uploaded' is never stored here
  appearance_version: number          // the look it was painted from
  painted_at: Date
}
```

| Who | Where `portrait` and `voice` live |
|---|---|
| World cast and companion | the cast entry on `world_templates.seed_cast[]` / `protagonist` |
| Character born during play | the save's codex card (`characters` collection) |
| A world character repainted in one save | the save's card, overriding the world's for that save only |
| The player's persona | `personas` |

- **Looks.** The card's `appearance` text is the source. On a lasting change backed by a quoted sentence, the existing codex extractor rule (`character-codex.service.ts:610`, today fill-only) updates the save's `appearance` and bumps `appearance_version`. A portrait whose `appearance_version` is behind is **out of date**: it is repainted on the next tap that needs it, never on its own.
- **Unknown looks** are filled in at the first portrait, consistent with what's known, and written to `appearance` (canon from then on; the narrator already reads it).
- **Repaints** use the previous portrait as the reference image.
- **Image source.** Every image record gets `source`. Existing covers have none and count as `uploaded`.

### 3.1 Art style: the world's, never one house style

A world's media is drawn the way its cover is: an anime world gets anime portraits, a noir world gets noir ones (owner, Oct 16 2026). Until this lands, every image is forced into one painted style.

- **Source: the world's locked `narrative_style`**, the same key the cover generator already maps to an art style (`STYLE_HINT` in `image.service.ts`). There is one table for covers and media. No new authored field, and no backfill: every world in the dev database has a style (Oct 16 2026).
- **Look, not mood.** Each table entry splits into `render` (how it is drawn, e.g. "vibrant anime illustration, clean linework") and `mood` (atmosphere, e.g. "warm intimate lighting"). Covers keep `render + mood`, so their text stays byte-identical. **Media uses `render` only**, because a mood is atmosphere, not a drawing style. When a render itself carries a mood word, the entry gets a media-only `media` render that replaces it (today only `flirty`, whose render says "sensual"; `88354b3`). The mood comes from the moment itself. A missing or unknown key falls back to the painted default: "semi-realistic cinematic digital painting, warm rim light".
- **The template's style, never the save's.** A player may override the narrative voice for one conversation. That never changes the art style, because portraits are shared by every save of a world.
- **Where it goes:**
  - the portrait suffix;
  - the picture prompt;
  - the still template (it replaces the house style in §3A.3);
  - the director's `art_style` input.

  The line shot inherits the style from the portrait it animates.
- **Stored on the portrait.** `portrait.art_style` holds the style key it was painted in. A portrait whose key differs from the world's current key is **out of date**, exactly like a behind `appearance_version`. It repaints on the next tap that needs it, never on its own, using the old portrait as the identity reference with the new style in the prompt. The cache key includes the style key.
- **Persona portraits** follow the persona across worlds. Each one is kept **per style key** (`personas.portraits[<key>]`) and painted on the first tap in a world of that style. The first painting in a new style counts against the free portrait allowance. (owner, Oct 16 2026)

---

## 3A. The video: a scene cut

"See them say it" is a short cinematic cut (~10–12 s), not only a talking head (owner, Oct 16 2026). The narration becomes what is seen; the quoted line is what is heard.

| Shot | Length | Made from | Rules |
|---|---|---|---|
| **Opening** | 3–4 s | the turn's narration **before** the line | Wide or detail framing: the place, hands, a silhouette, the character from behind. **No face close-up**: video models drift on faces (Seedance did in the probe). Image-to-video from **its own wide still** (one image call, ≈ $0.009), drawn in the world's art style (§3.1) with no face close-up, so the look carries over. **Never start from a portrait or a face-framed picture:** the scene probe showed a close-up first frame cannot be prompted into a wide shot (models duplicated the woman or reframed abruptly) |
| **The line** | the clip's length (≈ 3–8 s) | the character's portrait + **the exact voice clip the player heard** | The talking portrait (§9.1). The only shot with the face up close |
| **Closing** (optional) | 2–3 s | the narration **after** the line | Same framing rules as the opening. Left out when nothing follows the line |

- The narration is not shown as text. Optional subtitles of the spoken line are a later choice.

### 3A.1 The director pass (premium)

The quality of the cut is decided by its direction, not by the video model (owner, Oct 16 2026). One call to a **premium director model** turns the turn into a structured shot plan.
- **The prompt and output schema are fixed:** `server/MEDIA_DIRECTOR_PROMPT.md`.
- The director writes a **continuity sheet** first: wardrobe, hair, props, time, weather, light direction, palette. Then the shots, each with:
  - framing and camera move from closed lists;
  - who is in the still;
  - the exact first-frame composition;
  - one subject action;
  - a verbatim evidence sentence from the narration.
- It also chooses the transitions and the voice lead-in (a J-cut), each from a closed list.
- **Model: chosen by a text-only bake-off inside `media-server-2`.** Three to four premium models are tried on the same fight, tense and intimate turns. The criteria:
  - rule compliance, scored by the §3A.2 validator;
  - the intimate turn is turned suggestive-but-not-explicit without refusing;
  - the owner's read of the plans.
- **Cost:** ≈ $0.01–0.04 per video, small next to the shots. Env `MEDIA_DIRECTOR_MODEL`.
- **Never cached across turns;** cached per video request.

### 3A.2 Validation in code: the probe's glitches, closed structurally

| Probe glitch | Closed by |
|---|---|
| A second woman appeared at the door (Grok, Veo) | `people_in_still` is the complete cast of the shot. The compiled motion prompt says "no other people appear". The post-check counts people on the first, middle and last frames, and the counts must match `people_in_still` |
| Abrupt reframe / foreground jump (Seedance) | Exactly one `camera_move` from the closed list. The motion is compiled as a change from the still, never a new composition |
| Costume drift (Wan: brown leather became a black jacket) | The continuity sheet's wardrobe is pasted verbatim into every still and motion prompt. The speaker's portrait is passed as a wardrobe/hair reference for the still. The post-check compares the last frame with the still |
| Face close-up drift | Scene framings exclude the face by construction. The post-check rejects a frame where a face fills more than a set share of the frame |
| Shot not about the story | `evidence` must be a verbatim sentence of the right narration span (opening from before the line, closing from after it). Otherwise that shot is dropped |

The validator also rejects:
- names that don't resolve to the speaker, a present card, or nobody;
- durations or transitions outside the lists;
- more than one scene shot per role.

A plan that fails gets **one** repair call with the validator's errors. Failing again, the cut degrades to the line shot alone.

### 3A.3 Compilation and assembly in code

- **Prompts are compiled, not written by the director.**
  - Per-model templates live in data, and code fills them from the plan.
  - Still prompt: the world's art style (§3.1) + continuity + framing + `still` + "no text".
  - Motion prompt: `camera_move` + `subject_action` + `environment_motion` + wardrobe + "no other people appear, no cut, the composition stays continuous".
- **Post-check on every scene shot**, a vision call on the first, middle and last frames:
  - the content check (§7);
  - the people count;
  - the wardrobe match;
  - no face close-up;
  - no text.

  A failure gets **one** retry on the fallback scene model (Wan, audio dropped). Failing again, the shot is dropped and the cut degrades. Retries are our cost, never an extra charge to the player.
- **Assembly** with ffmpeg, using the probe's recipe:
  - one resolution (720×1280) and 24 fps;
  - the director's transitions;
  - the J-cut lead-in;
  - our voice at full level;
  - the ambient bed under it at about −18 dB, never a voice;
  - loudness normalised to about −16 LUFS;
  - one shared colour grade across shots, so the HeyGen shot and the scene shots match;
  - the line shot trimmed to the voice plus a 0.3 s tail.
- **The whole plan is stored** with the `media_assets` record (plan, compiled prompts, check results), so a bad cut can be diagnosed.
- Every model call goes through the content gates (§7): the plan is checked before any generation, and every shot after.
- **Degrades, never breaks.** If a scene shot fails or is rejected, the cut is delivered with the remaining shots (at minimum the line shot) and the player is charged one video. If the line shot fails, the whole video is `media_failed` and the slot is released.
- **One video counts once**, whatever the shot count.

## 4. The player's actions (UX)

| Action | Where | What happens |
|---|---|---|
| **Hear a line** | Tap a spoken line in the prose | The line shimmers softly while it loads, then plays. Tapping again stops it. A small floating player appears only while something is playing |
| **See them say it** | Long-press a spoken line → small menu "Hear it · See them say it" | A compact placeholder ("Taking shape…") appears under the turn and becomes the video when it's ready |
| **Picture this** | The turn's existing long-press menu (beside Rewind / Copy / Edit) | Same placeholder, then the picture |
| **Give yourself a face** | Offered once, the first time a picture would include the player; also on the persona screen | Paints the player's portrait |
| **Repaint** | A card's portrait (codex / persona) | Repaints that face (counts as one picture) |

- Spoken lines get no new chrome. Discovery is a one-time coachmark through the existing guide system: "Tap what they say to hear it."
- A finished picture or video is a single collapsed thumbnail under its turn. Tapping opens it full screen; collapsed is the default, so long chats stay readable.
- Media is never auto-played (except a video the player just asked for, which plays once, muted until tapped).

---

## 5. API

All under the existing auth. The player must own the save.

```
POST /instances/:instanceId/events/:eventId/lines/:lineId/voice
  → 200 { status: 'ready', url, duration_ms }                  // cached or done in time
  → 202 { status: 'queued', media_id }                          // arrives via media_ready
  → 422 { status: 'unvoiceable' }                               // §2.2 failed, or mature without opt-in
  → 429 { status: 'resting', kind: 'voice', resets_at }

POST /instances/:instanceId/events/:eventId/picture
POST /instances/:instanceId/events/:eventId/lines/:lineId/video
  → 202 { status: 'queued', media_id } | 200 { status: 'ready', url } | 429 resting

POST /instances/:instanceId/characters/:characterId/portrait   // repaint
POST /personas/:personaId/portrait                             // "give yourself a face"
  → 202 queued | 429 resting

GET  /media/usage → { voice: {used, limit}, picture: {...}, video: {...}, portrait: {...}, resets_at }
```

`resets_at` is the next UTC midnight ISO, the same as `STORY_RESTING`.

---

## 6. Socket messages (existing `user:<id>:events` channel)

```
{ type: 'spoken_lines', instanceId, eventId, lines: [{line_id, start, end}] }   // after the stream
{ type: 'media_ready', instanceId, eventId, media_id, kind: 'voice'|'picture'|'video'|'portrait', url, line_id? , character_id? }
{ type: 'media_unseen', instanceId, eventId, media_id, kind }                    // §7 rejection
{ type: 'media_failed', instanceId, eventId, media_id, kind }                    // provider failure, limit released
```

`spoken_lines` is also included on the event in `GET` instance/event payloads, so a reload shows the same lines.

A public line carries only `{line_id, start, end}`, plus `voice: {status: 'unvoiceable'}` when refused, plus `filmable: false` once attribution has found its speaker has no card (§2.2). The app hides "See them say it" for a `filmable: false` line. Before the first tap nothing is known, so it is offered, and a video request for an uncarded speaker answers 422 `{ status: 'unvoiceable' }`-style with the calm line *"This moment stays unseen."*, uncharged.

**Feature switch on the wire.** While `MEDIA_ENABLED` is off, the server sends no `spoken_lines` (no socket message, no payload field) and `GET /media/usage` returns 404. The app shows media affordances only when a turn has `spoken_lines` (voice, video) or `/media/usage` answered 200 this session (Picture this, portraits). Switching media off therefore hides it without an app release.

---

## 7. Safety (pictures, portraits, video)

1. **Visual description.** A cheap model turns the moment into a description that stays within the line below: who is present (by portrait reference), framing, setting, mood. An intimate moment may be shown as sensual and suggestive (an embrace, a kiss, closeness, revealing-but-covered clothing), never nude and never a sexual act.
2. **Gate before generation.** A moderation check on the description. The following → `media_unseen`:
   - full nudity (exposed genitals, female nipples, buttocks);
   - a sexual act;
   - anyone who looks underage in a sensual or suggestive context;
   - real people;
   - graphic gore.

   Suggestive content between adults is **allowed**. This is a check, not a prompt instruction. The line is written once, as data (`MEDIA_VISUAL_POLICY` in `safety.ts`), and the description, moderation and vision prompts are all built from it. Each blocked item has its own reason code; looking underage in a suggestive context is `underage_suggestive`, judged from the picture, never from a stated age (`1824035`).
3. **Gate after generation.** A vision check on the image, or on sampled frames of a video (first, middle, last), against the same line as the gate before. Failing → `media_unseen`, and the file is deleted.
4. **References are only `generated` / `authored` portraits.**
5. **Never charged when unseen.** The limit is released. Copy: *"This moment stays unseen."*
6. Every rejection is logged (kind, stage, reason code, no image) to a `media_rejections` count for review.

Voice skips 1–3 and follows the mature opt-in (§2.3). The existing hard input guard (minors, incest) already stops those turns before any prose exists.

---

## 8. Limits

Daily per player, UTC midnight reset, the same for every tier during the promo (env-configurable):

| Kind | Default | Env | Resting copy |
|---|---|---|---|
| Voice lines | 100 | `MEDIA_DAILY_VOICE` | "Their voices rest until tomorrow." |
| Pictures (incl. repaints) | 15 | `MEDIA_DAILY_PICTURE` | "The painter rests until tomorrow." |
| Videos | 2 | `MEDIA_DAILY_VIDEO` | "The visions rest until tomorrow." |
| New portraits (free) | 20 | `MEDIA_DAILY_PORTRAIT` | (silent; the requested media waits until tomorrow) |

- **Accounting** mirrors ink reservations: a `media_usage` row is reserved on request, settled on delivery and released on failure or rejection. Counts come from rows, so a free cached replay costs no slot (a cache hit never reserves).
- Spend per call is logged from the OpenRouter generation id.
- A global kill switch `MEDIA_ENABLED`, **on by default** (owner, Oct 16 2026: no players yet, production prep). Set `MEDIA_ENABLED=false` to switch everything off, or `MEDIA_<KIND>_ENABLED=false` for one kind.

---

## 9. Server shape

- **`src/ai/`**: `tts.ts` (exists) gains PCM→m4a when the model returns PCM. New `image.ts` (generate with references) and `video.ts` (submit job, poll, fetch).
- **`src/services/media/`**: request handling, limits, cache keys, voice casting, portrait resolution, safety gates.
- **Media queue**: new BullMQ `media` queue in `src/queues/index.ts`; `worker/processors/media.processor.ts` with its own concurrency, so media never delays `generation`. Voice jobs get top priority.
- **Collections**: `media_assets` (player, instance, event, kind, line_id, character_id, source, model, cost_usd, status, asset_key, created_at), `media_usage` (reservation rows). No explicit `null` under any unique+sparse index.
- **Deletion**: both collections join the shared purge list in `deletion.service.ts`, with S3 object deletion. Rewinding deletes media of the removed events. Deleting a world deletes its cast portraits.
- **Audits**: `audit:media`: detection rules (unclosed quotes, apostrophes, nested quotes, scare quotes, paragraph runaway), the verbatim-substring rule, cache-hit-reserves-nothing, unseen-is-free, mature-voice gate, uploaded-never-a-reference, limit reset. Every check shown to bite.

### 9.1 Models

| Kind | Model | Status | Env |
|---|---|---|---|
| Voice | `elevenlabs/eleven-v4-turbo` | **chosen** (owner, Oct 16 2026) | `MEDIA_VOICE_MODEL` |
| Line attribution (§2.2) | the existing cheap structured-call model | as today | — |
| Portrait / picture / scene still | `black-forest-labs/flux-3-image-klein` ($0.009 per 9:16 image at 768×1360; up to 10 face references; returns **JPEG**). It replaced `flux.2-klein-4b` ($0.015) because Flux 2 Klein's own filter refused ordinary scarred faces as "graphic violence" while Flux 3 Klein accepted them, and it matched the painted house style best in a side-by-side (`.agents` test, Oct 16 2026, $0.14). It also draws anime well when the prompt asks for it. Black Forest Labs still filters, so a refusal goes down the provider-refusal path (§7). Media images send `safety_tolerance` 2, the provider default; a live check showed no gain at 3 (`1824035`) | **chosen** (owner, Oct 16 2026) | `MEDIA_IMAGE_MODEL`, `MEDIA_IMAGE_SAFETY_TOLERANCE` (integer 0–5) |
| Director (§3A.1) | `openai/gpt-6.1-sol`: valid on every bake-off turn at about $0.013 per plan; Opus 5.5 was also valid at about 3× the price | **chosen** (bake-off, Oct 16 2026) | `MEDIA_DIRECTOR_MODEL` |
| Moderation gate (§7) | `openai/gpt-4o-mini`, structured verdict, about $0.0001 per check | chosen | `MEDIA_MODERATION_MODEL` |
| Vision gate (§7) | `google/gemini-2.5-flash-lite`, about $0.0002 per image | chosen | `MEDIA_VISION_MODEL` |
| Talking video (the line shot) | `heygen/avatar-iv` (~$0.05/s; it kept our exact recording, 0.985 correlation, and the painted face) | **chosen** (owner, Oct 16 2026) | `MEDIA_VIDEO_MODEL` |
| Scene shots (§3A) | `x-ai/grok-imagine-video-1.5-lite`, 480p, about $0.02/s plus $0.01 per input image; it had the best painted look and was cheapest. **Fallback `alibaba/wan-3.0`**, used automatically when Grok fails: a scene shot has no voice to keep consistent. Wan's own audio track is **always dropped**, because it returned audio even with `generate_audio:false` and the voice check flagged it | **chosen** (owner, Oct 16 2026) | `MEDIA_SCENE_MODEL`, `MEDIA_SCENE_FALLBACK_MODEL` |

API shapes the probe established:
- **Images:** `POST /api/v1/images` with `prompt`, `aspect_ratio`, `n`, `input_references`.
- **Video:** `POST /api/v1/videos`, `GET /videos/{id}`, `GET /videos/{id}/content`.
  - HeyGen rejects `first_frame`. It takes the portrait and the audio as `input_references`, and the audio must be at a reachable HTTPS URL. Our CDN URL serves.
  - Seedance swapped in a different voice and drifted to a photoreal face. Veo invents its own voice. Neither is a fallback.

## 10. App shape

- `play/`: tappable spoken lines (from `spoken_lines` offsets), the inline shimmer and floating player, the long-press additions, collapsed media thumbnails under turns, the resting/unseen copy.
- `codex` card and persona screen: portrait display and repaint.
- One coachmark in the guide system.

## 11. Tasks (after the probe)

| Id | Lane | Paths |
|---|---|---|
| media-server-1: detection, voice, limits | Sol (touches billing-style accounting and content gating) | `everlore-server/src/**`, `worker/**`, `scripts/**` |
| media-server-2: portraits, pictures, video, safety | Sol (content safety) | same, after task 1 |
| media-app: lines, menus, thumbnails, portraits | agy | `everlore/lib/features/play/**`, codex, persona, `test/**` |
