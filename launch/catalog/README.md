# Launch catalog: 40 new worlds

Agreed with the owner Oct 22 2026 (`DECISIONS.md`, "Launch catalog: 40 new worlds"). These 40 join the 12 already live, for 52 in all. Each is a **GM world** or a **character**; there are no sentient worlds.

## How a world is written

One JSON file per world, `<batch>/<slug>.json`, holding exactly the body `POST /templates` takes, plus two fields the seed strips before sending:

- `slug`: the file's own key, so the seed is idempotent (it looks the title up under the Everlore account before creating).
- `cover`: the cover file name, `cover_<slug>.png`, in the Codex drop folder `world assets/chat-covers/`.

Shapes follow what the app itself sends (`create_character_cubit.dart`, `forge_world_cubit.dart`):

| | GM world | Character |
|---|---|---|
| `kind` / `is_sentient` | `world` / `false` | `character` / `true` |
| `protagonist` | none: the player is "you", in their own persona | `{name, persona, appearance}`, the character the player talks to |
| `seed_prompt` | how the narrator runs the world | `You are <Name>. <tagline>.` then the persona |
| `global_lore` | the world: places, powers, people (the cast is extracted from it) | the character's backstory and the people around them |
| `opening_line` | the first scene, second person | the character's greeting |
| `base_stats_template` | at least one stat | `{}` |
| `image_prompt` | the visual core for the cover; the Codex brief adds the style tail | the character's portrait |

Limits (`FIELD_LIMITS`): title ≤80, description ≤600, seed_prompt 10–2500, global_lore ≤3000, opening_line ≤600, image_prompt ≤1400, protagonist name ≤80 and text ≤400.

Rules for every file:
- **Every character is an adult**, and says so in the lore. Nothing is set in a school below university.
- No vendor, model or mechanic words, and no "AI", in anything a player reads (title, description, opening).
- The description is what the shelf and the website show, so it sells the premise in two sentences.
- Romance worlds are `is_nsfw_capable: true`; the engine's content bounds apply as everywhere.

## The slate

Families are the app's interest families (`STYLE_FAMILY` in `narrative-styles.ts`). Live now: dark 8, epic 1, romance 1, modern 1, cozy 1.

### Romance (+11) · batch `romance` · first
| Slug | Title | Kind | Voice |
|---|---|---|---|
| the-dons-debt | The Don's Debt | character | dark_romance |
| lord-of-nightfall-manor | Lord of Nightfall Manor | character | dark_romance |
| moonbound | Moonbound | world | dark_romance |
| the-fae-bargain | The Fae Bargain | world | dark_romance |
| the-season-at-ashcombe | The Season at Ashcombe | world | regency |
| yuna-only-has-eyes-for-you | Yuna Only Has Eyes for You | character | yandere |
| the-prince-you-were-sent-to-kill | The Prince You Were Sent to Kill | character | dark_romance |
| your-bodyguard | Your Bodyguard | character | flirty |
| the-siren-of-blackreef | The Siren of Blackreef | character | dark_romance |
| the-duke-of-ravensworth | The Duke of Ravensworth | character | regency |
| the-angel-who-fell-for-you | The Angel Who Fell for You | character | flirty |

### Epic adventure (+10)
Summoned to the Wrong Kingdom (world, litrpg) · The Hundredth Floor (world, litrpg) · Ember Academy (world, shonen) · Wings of Valdris (world, epic_fantasy) · Salt and Cannon (world, epic_fantasy) · The Lotus Throne (world, epic_fantasy; inspired by Indian epics, no real deities or scripture) · Starfall Frontier (world) · The Long Winter Saga (world, epic_fantasy) · Kaelith, the Last Dragon (character) · Ren, Your Rival (character, shonen)

### Modern everyday (+8)
Backstage at Starlight (world, kdrama) · Mr. Han, Your Boss (character, kdrama) · The Wedding Week (world, romcom) · Roommate Wanted (character, romcom) · Summer in Saltmere (world, slice_of_life) · Ten Years Later (world, romcom) · Startup Nights (world, modern_casual) · Your Best Friend Since Forever (character, slice_of_life)

### Cozy and playful (+7)
The Little Potion Shop (world, cozy_comfort) · Hollowbrook Farm (world, cozy_comfort) · The Demon Lord's Day Off (world, chaotic_comedy) · The Kitsune Next Door (character, anime) · Mister Biscuit (character, whimsical) · The Inn at World's End (world, cozy_comfort) · Pip, the Overly Helpful Ghost (character, chaotic_comedy)

### Dark and atmospheric (+4)
The Last Safe House (world, horror) · The House on Hollow Lane (world, horror) · The Gaslight Detective (world, noir) · Wasteland Radio (world, grimdark)

Totals after: dark 12, epic 11, romance 12, modern 9, cozy 8. 17 characters, 23 worlds.

## Pipeline per batch
1. Write the files (orchestrator), owner reviews.
2. Covers from the Codex app: `.agents/assets/chat-covers-<batch>.md` (`WORLD_CREATION_PLAYBOOK.md` §4B).
3. Seed: dev first, then prod under the official **Everlore** account (AWS login rule).
4. World pages on everloreapp.com, then `npm run indexnow`.

## Status
| Batch | Files | Covers | Seeded | Pages |
|---|---|---|---|---|
| romance | drafted Oct 22 | – | – | – |
