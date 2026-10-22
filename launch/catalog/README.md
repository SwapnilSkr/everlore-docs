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
- **Every world is `is_nsfw_capable: true`** (owner, Oct 22 2026, `DECISIONS.md`): the player's own toggle decides. The engine's content bounds apply as everywhere.

## The slate (about 100 new worlds)

**Audience (owner, Oct 22 2026): 8 of every 10 worlds are aimed at men, 2 at women.** In a world aimed at men the lead or love interest is a woman, and the player may be written as a man where the trope needs it (a "fake boyfriend"); otherwise the player stays "you". The player's own persona always wins.

| Family | New worlds |
|---|---|
| Romance | 25 (20 for men, 5 for women) |
| Epic adventure | 25 (20 for men, 5 for women) |
| Dark and atmospheric | 20 (16 for men, 4 for women) |
| Horror | 25 (20 for men, 5 for women) |
| Modern everyday | 15 (12 for men, 3 for women), live |
| Cozy and playful | ~15, paused (owner, Oct 22: Explore genres come first) |

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

### Epic adventure (25: 20 for men, 5 for women; 12 GM worlds, 13 characters; 7 anime covers)
Agreed with the owner Oct 22 2026. NSFW-capable since Oct 22 (owner: every world is; first published without it).

| Slug | Title | Kind | Voice | Aimed at |
|---|---|---|---|---|
| blood-and-sand | Blood and Sand | GM world | grimdark | men |
| captain-of-the-sky-fleet | Captain of the Sky Fleet | character | epic_fantasy | women |
| dungeon-lord | Dungeon Lord | GM world | chaotic_comedy | men |
| ember-academy | Ember Academy | GM world | shonen | men |
| freya-shieldmaiden-of-the-north | Freya, Shieldmaiden of the North | character | epic_fantasy | men |
| kaelith-the-last-dragon | Kaelith, the Last Dragon | character | epic_fantasy | women |
| lyra-the-elf-ranger | Lyra, the Elf Ranger | character | epic_fantasy | men |
| master-kenjis-last-student | Master Kenji's Last Student | character | default | men |
| mira-the-mage-who-summoned-you | Mira, the Mage Who Summoned You | character | anime | men |
| ren-your-rival | Ren, Your Rival | character | shonen | men |
| sable-the-mercenary-who-owes-you | Sable, the Mercenary Who Owes You | character | tsundere | men |
| salt-and-cannon | Salt and Cannon | GM world | epic_fantasy | men |
| starfall-frontier | Starfall Frontier | GM world | default | men |
| summoned-to-the-wrong-kingdom | Summoned to the Wrong Kingdom | GM world | anime | men |
| the-exiled-prince-you-travel-with | The Exiled Prince You Travel With | character | epic_fantasy | women |
| the-exiled-princess-and-her-last-knight | The Exiled Princess and Her Last Knight | character | epic_fantasy | men |
| the-hundredth-floor | The Hundredth Floor | GM world | litrpg | men |
| the-last-ronin | The Last Ronin | GM world | epic_fantasy | men |
| the-long-winter-saga | The Long Winter Saga | GM world | grimdark | men |
| the-lotus-throne | The Lotus Throne | GM world | epic_fantasy | men |
| the-sorcerer-who-took-you-in | The Sorcerer Who Took You In | character | dark_academia | women |
| the-warlords-road | The Warlord's Road | GM world | grimdark | men |
| valka-the-warchiefs-daughter | Valka, the Warchief's Daughter | character | flirty | men |
| wings-of-valdris | Wings of Valdris | GM world | epic_fantasy | women |
| zephyr-the-thief-who-stole-your-quest | Zephyr, the Thief Who Stole Your Quest | character | flirty | men |

### Dark and atmospheric (20: 16 for men, 4 for women; 10 GM worlds, 10 characters)
Agreed with the owner Oct 22 2026. NSFW-capable since Oct 22 (first published without it). Every cover painted (no anime). Voices: horror 7, grimdark 6, noir 3, cyberpunk 2, dark academia 2.

| Slug | Title | Kind | Voice | Aimed at | Premise |
|---|---|---|---|---|---|
| the-last-safe-house | The Last Safe House | GM world | horror | men | zombie survival: you hold the last safe house in a dead city, and every night someone new knocks |
| the-house-on-hollow-lane | The House on Hollow Lane | GM world | horror | men | haunted house: you inherit a house that remembers everyone who died in it |
| the-gaslight-detective | The Gaslight Detective | GM world | noir | men | Victorian detective: fog, gaslight, and a killer who leaves you letters |
| wasteland-radio | Wasteland Radio | GM world | grimdark | men | post-apocalyptic: you run the last radio station, and every broadcast draws someone to your door |
| the-hunt-at-varnholm | The Hunt at Varnholm | GM world | grimdark | men | monster hunter: a plague city where the beasts were people last week |
| ward-thirteen | Ward Thirteen | GM world | horror | men | night-shift horror: you're the new night guard at a facility that keeps things that shouldn't exist |
| zero-day | Zero Day | GM world | cyberpunk | men | cyberpunk heist: one job inside a megacorp tower, a crew you don't trust |
| the-drowned-town | The Drowned Town | GM world | horror | men | cosmic horror: a fog-bound fishing town, a cult, and something under the harbour |
| dead-mans-hand | Dead Man's Hand | GM world | grimdark | men | weird west: a frontier town where the dead don't stay buried |
| the-hollow-society | The Hollow Society | GM world | dark_academia | women | dark academia: an old university's secret society invites you in, and members keep vanishing |
| vesna-the-vampire-who-spared-you | Vesna, the Vampire Who Spared You | character | horror | men | vampire: she should have killed you; now she wants to know why she didn't |
| detective-iris-kane-your-partner | Detective Iris Kane, Your Partner | character | noir | men | noir detective: your homicide partner in a rain-soaked city, hiding a case of her own |
| the-woman-in-the-lighthouse | The Woman in the Lighthouse | character | horror | men | ghost mystery: the keeper of a storm-cut island, and you're not sure she's alive |
| morgana-the-cults-high-priestess | Morgana, the Cult's High Priestess | character | grimdark | men | cult infiltration: you've joined her order to bring it down, and she's starting to suspect you |
| jinx-the-fixer-who-owns-your-debt | Jinx, the Fixer Who Owns Your Debt | character | cyberpunk | men | cyberpunk: you owe her everything, and she's calling it in one job at a time |
| ylva-the-wolf-who-found-you | Ylva, the Wolf Who Found You | character | grimdark | men | werewolf: she dragged you out of the snow; the pack wants you gone |
| sister-agnes-the-exorcist | Sister Agnes, the Exorcist | character | horror | men | exorcism: a hard-bitten nun drags you into her worst case yet |
| professor-vale-who-teaches-the-forbidden | Professor Vale, Who Teaches the Forbidden | character | dark_academia | women | dark academia: a brilliant, dangerous professor takes you as his only student |
| cassian-the-monster-hunter | Cassian, the Monster Hunter | character | grimdark | women | monster hunter: a scarred hunter hired to protect you from whatever is following you |
| the-detective-who-thinks-you-did-it | The Detective Who Thinks You Did It | character | noir | women | noir: he's certain you're the killer, and he won't stop coming round |

### Horror (25: 20 for men, 5 for women; 12 GM worlds, 13 characters)
Agreed with the owner Oct 22 2026. Asked for by the owner Oct 22 2026 as its own batch after Dark. All `horror` voice; every cover painted. It avoids the horror ideas the Dark batch already uses (zombies, a haunted house, the night guard, cosmic horror, a vampire, a ghost on an island, exorcism, a werewolf, weird west). NSFW-capable since Oct 22 (first published without it).

| Slug | Title | Kind | Voice | Aimed at | Premise |
|---|---|---|---|---|---|
| the-woods-behind-kettle-creek | The Woods Behind Kettle Creek | GM world | horror | men | creature in the woods: hikers vanish, and something out there learns to use their voices |
| deep-station-nadir | Deep Station Nadir | GM world | horror | men | deep-sea horror: a research station on the ocean floor loses contact with the surface |
| the-derelict | The Derelict | GM world | horror | men | space horror: your salvage crew boards a ship missing for thirty years, and it is warm inside |
| the-midnight-motel | The Midnight Motel | GM world | horror | men | a roadside motel you can't drive away from, where new guests check in every night |
| the-lake-house-weekend | The Lake House Weekend | GM world | horror | men | slasher: a weekend with old friends, a masked stranger, and no signal |
| the-harvest-at-ember-hollow | The Harvest at Ember Hollow | GM world | horror | men | folk horror: a remote village welcomes you to its harvest festival a little too warmly |
| the-grand-halloway-hotel | The Grand Halloway Hotel | GM world | horror | men | a grand old hotel whose floors rearrange at night, and the guests on the thirteenth never leave |
| the-last-train | The Last Train | GM world | horror | men | a night train that never stops, and the passengers don't know they're dead |
| the-sleepwalkers-of-morrow | The Sleepwalkers of Morrow | GM world | horror | men | a town where everyone dreams the same nightmare, and you are in it |
| the-loop | The Loop | GM world | horror | men | time-loop survival: the same night, the same killer, and you remember every death |
| haunt-hunters | Haunt Hunters | GM world | horror | men | you host a ghost-hunting show, and this week's location is real |
| the-dollhouse | The Dollhouse | GM world | horror | women | an antique dollhouse that copies your home and shows what happens tomorrow |
| ilsa-the-woman-in-the-mirror | Ilsa, the Woman in the Mirror | character | horror | men | the woman trapped in your antique mirror wants out, and needs your help |
| hana-the-ghost-in-your-apartment | Hana, the Ghost in Your Apartment | character | horror | men | J-horror: your cheap new apartment comes with its last tenant, and she's not leaving |
| dr-elena-voss-the-quarantine-doctor | Dr. Elena Voss, the Quarantine Doctor | character | horror | men | outbreak: the two of you are sealed inside a hospital with something spreading |
| ruby-the-final-girl | Ruby, the Final Girl | character | horror | men | the only survivor of a massacre says the killer is back, and you're the only one who believes her |
| bryony-the-bog-witchs-daughter | Bryony, the Bog Witch's Daughter | character | horror | men | folk horror: the village fears her, she saved your life, and her mother wants something for it |
| clara-the-ventriloquist | Clara, the Ventriloquist | character | horror | men | her dummy talks when she doesn't, and it has taken an interest in you |
| sera-the-other-survivor | Sera, the Other Survivor | character | horror | men | a plane down in the mountains, two survivors, and something circling the wreck |
| wren-the-medium | Wren, the Medium | character | horror | men | a psychic who hears the dead around you, and they are all saying the same thing |
| nadia-the-profiler | Nadia, the Profiler | character | horror | men | a criminal profiler hunting a serial killer whose next target is you |
| silas-the-gravedigger | Silas, the Gravedigger | character | horror | women | he keeps the town's dead in their graves, and one of them is asking for you |
| dorian-the-painter | Dorian, the Painter | character | horror | women | a reclusive painter whose portraits come alive, and he wants to paint you |
| ezra-the-man-who-doesnt-sleep | Ezra, the Man Who Doesn't Sleep | character | horror | women | he hasn't slept in twenty years to keep something away, and now it's found you |
| gideon-the-cult-survivor | Gideon, the Cult Survivor | character | horror | women | he escaped a cult, it wants him back, and you're the one hiding him |

### Modern everyday (15: 12 for men, 3 for women; 7 GM worlds, 8 characters)
Agreed with the owner Oct 22 2026. Everyday life now, with a love interest at the centre of most. All NSFW-capable, like every world. Every cover painted. Voices: slice of life 6, romcom 4, kdrama 3, modern casual 2. It skips ideas already live: a startup (Founders' Floor), a campus (After Hours at Halcyon U, the campus bully), fake dating and the heiress bodyguard (Romance).

| Slug | Title | Kind | Voice | Aimed at | Premise |
|---|---|---|---|---|---|
| the-wedding-week | The Wedding Week | GM world | romcom | men | you're best man at your oldest friend's wedding, and the maid of honour is your ex |
| ten-years-later | Ten Years Later | GM world | slice_of_life | men | the hometown reunion, and the one who got away is back and single |
| summer-in-saltmere | Summer in Saltmere | GM world | slice_of_life | men | one summer running your uncle's surf shop in a seaside town |
| backstage-at-starlight | Backstage at Starlight | GM world | modern_casual | men | you manage a rising band on its first tour, and the lead singer keeps you up all night |
| the-all-night-diner | The All-Night Diner | GM world | slice_of_life | men | you run a 24-hour diner, and every regular after midnight brings a story |
| second-chance-garage | Second Chance Garage | GM world | slice_of_life | men | you inherit your late father's failing garage and the town that comes with it |
| one-year-in-seoul | One Year in Seoul | GM world | kdrama | women | a year-long job in Seoul, a neighbour who won't stop helping, and a rival who won't stop competing |
| maya-your-new-roommate | Maya, Your New Roommate | character | romcom | men | you answered a "roommate wanted" ad, and she has a lot of house rules |
| jess-your-best-friend-since-forever | Jess, Your Best Friend Since Forever | character | slice_of_life | men | friends since you were kids, and lately she's been acting different around you |
| ji-woo-the-idol-next-door | Ji-woo, the Idol Next Door | character | kdrama | men | a burned-out idol hiding from the world in the flat across the hall |
| riley-your-personal-trainer | Riley, Your Personal Trainer | character | romcom | men | a relentless trainer who's taken your "get in shape" goal as a personal mission |
| sofia-the-bartender | Sofia, the Bartender | character | modern_casual | men | the bartender at your corner bar knows everyone's secrets, and wants to hear yours |
| lena-the-neighbour-downstairs | Lena, the Neighbour Downstairs | character | slice_of_life | men | a newly divorced night-shift nurse downstairs, a flooded ceiling, and a friendship that grows from favours (changed from a single mother so no child is in the cast) |
| mr-han-your-boss | Mr. Han, Your Boss | character | kdrama | women | the cold heir who runs your company hires you as his personal assistant |
| theo-the-grumpy-bookshop-owner | Theo, the Grumpy Bookshop Owner | character | romcom | women | you take a job at his failing bookshop, and he hates everything about it, except you |

### Other families
To be planned one at a time with the owner, at 8:2. The Oct 22 first draft (written before the 8:2 rule, so it will be re-cut toward male players):
- Cozy and playful: The Little Potion Shop, Hollowbrook Farm, The Demon Lord's Day Off, The Kitsune Next Door, Mister Biscuit, The Inn at World's End, Pip the Overly Helpful Ghost.

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
| romance | 25 written Oct 22, agreed | 25 done Oct 22 | **25 of 25 live** Oct 22 under Everlore (owner `6ad90def8ab4aac77be04ae4`) | | 25 live on everloreapp.com |
| epic | 25 written Oct 22, agreed | 25 done Oct 22 | **25 of 25 live** Oct 22 under Everlore | 25 live on everloreapp.com (`b0ece0a`) |
| dark | 20 written Oct 22, agreed | 20 done Oct 22 | **20 of 20 live** Oct 22 under Everlore | 20 live on everloreapp.com (`5a41ff2`) |
| horror | 25 written Oct 22, agreed | 25 done Oct 22 (2 redone after review: an extra hand; a killer's mask too close to film ones) | **25 of 25 live** Oct 22 under Everlore | 25 live on everloreapp.com (`e7d0bae`) |
| modern | 15 written Oct 22, agreed | 15 done Oct 22, all passed close review first time | **15 of 15 live** Oct 22 under Everlore; the same run switched adult content on for the 70 Epic, Dark and Horror worlds, and the 10 pre-catalog worlds were switched by a flag write (0 published worlds now have it off) | 15 live on everloreapp.com (`a29f211`) |
