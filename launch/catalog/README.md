# Launch catalog: about 100 new worlds

Agreed with the owner Oct 22 2026 (`DECISIONS.md`, "Launch catalog"). About 100 new worlds join the 12 already live. Each is a **GM world** or a **character**; there are no sentient worlds.

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

## The slate (about 100 new worlds)

**Audience (owner, Oct 22 2026): 8 of every 10 worlds are aimed at men, 2 at women.** In a world aimed at men the lead or love interest is a woman, and the player may be written as a man where the trope needs it (a "fake boyfriend"); otherwise the player stays "you". The player's own persona always wins.

| Family | New worlds |
|---|---|
| Romance | 25 (20 for men, 5 for women) |
| Epic adventure | ~25 |
| Dark and atmospheric | ~20 |
| Modern everyday | ~15 |
| Cozy and playful | ~15 |

Families are the app's interest families (`STYLE_FAMILY` in `narrative-styles.ts`). Live before the catalog: dark 8, epic 1, romance 1, modern 1, cozy 1.

### Romance (25) · batch `romance` · first, agreed Oct 22
| Slug | Title | Kind | Voice | Aimed at |
|---|---|---|---|---|
| lady-of-nightfall-manor | Lady of Nightfall Manor | character | dark_romance | men |
| moonbound | Moonbound | GM world | dark_romance | men |
| nova-your-netrunner-partner | Nova, Your Netrunner Partner | character | flirty | men |
| the-angel-who-fell-for-you | The Angel Who Fell for You | character | flirty | men |
| the-campus-bully-who-secretly-likes-you | The Campus Bully Who Secretly Likes You | character | tsundere | men |
| the-demon-queens-contract | The Demon Queen's Contract | character | flirty | men |
| the-dragon-who-claimed-you | The Dragon Who Claimed You | character | flirty | men |
| the-elf-who-hates-humans | The Elf Who Hates Humans | character | tsundere | men |
| the-fae-bargain | The Fae Bargain | GM world | dark_romance | men |
| the-fake-dating-deal | The Fake Dating Deal | character | flirty | men |
| the-heiress-you-guard | The Heiress You Guard | character | flirty | men |
| the-ice-empresss-consort | The Ice Empress's Consort | character | dark_romance | men |
| the-knight-commander | The Knight Commander | character | tsundere | men |
| the-mafia-queens-debt | The Mafia Queen's Debt | character | dark_romance | men |
| the-pirate-queens-prisoner | The Pirate Queen's Prisoner | character | flirty | men |
| the-princess-you-were-sent-to-kill | The Princess You Were Sent to Kill | character | dark_romance | men |
| the-siren-of-blackreef | The Siren of Blackreef | character | dark_romance | men |
| the-witch-who-cursed-you | The Witch Who Cursed You | character | flirty | men |
| your-maid-knows-too-much | Your Maid Knows Too Much | character | flirty | men |
| yuna-only-has-eyes-for-you | Yuna Only Has Eyes for You | character | yandere | men |
| lord-of-nightfall-manor | Lord of Nightfall Manor | character | dark_romance | women |
| the-dons-debt | The Don's Debt | character | dark_romance | women |
| the-duke-of-ravensworth | The Duke of Ravensworth | character | regency | women |
| the-prince-you-were-sent-to-kill | The Prince You Were Sent to Kill | character | dark_romance | women |
| the-season-at-ashcombe | The Season at Ashcombe | GM world | regency | women |

### Other families
To be planned one at a time with the owner, at 8:2. The Oct 22 first draft for them (written before the 8:2 rule, so most will be re-cut toward male players):
- Epic adventure: Summoned to the Wrong Kingdom, The Hundredth Floor, Ember Academy, Wings of Valdris, Salt and Cannon, The Lotus Throne (inspired by Indian epics, no real deities or scripture), Starfall Frontier, The Long Winter Saga, Kaelith the Last Dragon, Ren Your Rival.
- Modern everyday: Backstage at Starlight, Mr. Han Your Boss, The Wedding Week, Roommate Wanted, Summer in Saltmere, Ten Years Later, Startup Nights, Your Best Friend Since Forever.
- Cozy and playful: The Little Potion Shop, Hollowbrook Farm, The Demon Lord's Day Off, The Kitsune Next Door, Mister Biscuit, The Inn at World's End, Pip the Overly Helpful Ghost.
- Dark and atmospheric: The Last Safe House, The House on Hollow Lane, The Gaslight Detective, Wasteland Radio.

## Pipeline per batch
1. Write the files (orchestrator), owner reviews.
2. Covers from the Codex app: `.agents/assets/chat-covers-<batch>.md` (`WORLD_CREATION_PLAYBOOK.md` §4B).
3. Seed: dev first, then prod under the official **Everlore** account (AWS login rule).
4. World pages on everloreapp.com, then `npm run indexnow`.

## Spend
See `SPEND.md`, the running ledger, read from the provider's counter before and after every run.

## Status
| Batch | Files | Covers | Seeded | Pages |
|---|---|---|---|---|
| romance | 25 written Oct 22, agreed | 25 done Oct 22 | **25 of 25 live** Oct 22 under Everlore (owner `6ad90def8ab4aac77be04ae4`) | – |
