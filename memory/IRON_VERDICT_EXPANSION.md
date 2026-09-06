# The Iron Verdict — Expansion to three realms, 50 places, 40 characters

Supersedes the 16-location scope in the world bible §2. Everything in §§3–12 of
the bible (tiers, duels, travel, endings, romance, glory) is unchanged and
applies at the larger scale.

**The 15 Tier A assets already in flight remain valid.** Nothing below
invalidates them. The three Aldermere plates become three of seven; the five
sprites and five scenes are unchanged; fog and the sealed marker are used
harder than before, not differently.

---

## 1. Why three realms, and not just more map

Three kingdoms is only worth the money if each one *means* something. So each
realm is a different answer to one question — **how is law made?** — and the
conspiracy runs through all three.

| Realm | Law is made by | Attitude to the others |
|---|---|---|
| **Aldermere** (centre) | **Combat.** A Verdict in the Ring is final and cannot be appealed | Believes itself the only honest system: steel cannot be bribed |
| **Vaskir, the Cold Marches** (north) | **Oath.** Sworn word before the Thane-moot, backed by blood-price | Considers Verdicts barbarism — killing a man to settle a fence line |
| **Serevane, the Thousand Coins** (south) | **Purchase.** Every judgment has a listed price and a market | Considers both of the above theatre. It has been buying Aldermere's Verdicts for a century |

Serevane is the money behind the succession writ. Vaskir is where the evidence
survives, because they wrote down what Aldermere fought over. That gives the
player a reason to leave the duchy that is *plot*, not tourism.

### Map layout — one continuous vertical scroll

```
  Vaskir      2 plates   snow, pine, fjord      ~2048 x  4096
  ── seam: snow-capped mountain ridge ──
  Aldermere   3 plates   the duchy (ordered)    ~2048 x  6144
  ── seam: open sea ──
  Serevane    2 plates   islands, docks, canals ~2048 x  4096
                                          total ~2048 x 14336
```

Both seams are free. `plate_north`'s top edge is already specified as a
snow-capped ridge, which is exactly Vaskir's southern border. `plate_south`'s
bottom edge is already open sea, and Serevane is across the water — a sea-to-sea
seam needs no continuity at all. The seven plates were designed to stack before
we knew there would be seven.

---

## 2. The asset architecture that makes 50 places fit in 100 assets

Three ideas do all the work.

### Idea 1 — archetype sprites, reused

A village on the map is a village. Fourteen bespoke villages is money spent on
something no player will ever compare side by side. So the map draws from an
**archetype library**, varied at runtime by scale, mirroring, hue shift and
state, and only the places that carry story weight get a unique silhouette.

```
16 archetype sprites   village, hamlet, farmstead, watchtower, ruin, chapel,
                       longhall, mine head, quarry, wharf, warehouse, manor,
                       bridge, camp, gate, standing stones
12 hero sprites        the Verdict Ring, Sundial Market, Hollowforge,
                       Chainhouse, Marblegate, Sable Court, Iron Fane,
                       Hjarnmoot, Vaskhold, the Gilt Exchange, the Hundred
                       Docks, Blackreach Hold
```

28 sprites cover 50 locations. This is how painted strategy maps have always
worked, and nobody notices.

### Idea 2 — a scene is only for a place you can stand in

Of 50 locations, roughly half are ever entered. The rest are map presence:
named, discussed, travelled past, owned, fought over, given as a reward — and
never rendered at ground level. They cost one shared sprite each.

### Idea 3 — a portrait is only for a character with a face in the fiction

24 principals get bespoke art. The other 16 are recurring but framed — heard,
quoted, met at distance, or represented by their household. They earn codex
cards and bond meters from recurrence, exactly as the system already does.

### The budget

| Family | Count | Note |
|---|---|---|
| Terrain plates | 7 | 3 already ordered |
| Archetype sprites | 16 | reused across all 50 places |
| Hero sprites | 12 | 5 already ordered |
| Map kit | 3 | fog, sealed marker, ruin marker |
| Scenes | 26 | 5 already ordered |
| Duel stages | 2 | oath circle, harbour — the Ring scene covers the third |
| Character portraits | 22 | 24 principals, 2 already exist |
| **Total** | **88** | ~12 spare for awakened variants |

---

## 3. The fifty places

★ = already ordered in Tier A. ⬛ = enterable, needs a scene. ○ = map presence
only, uses an archetype sprite.

### Aldermere — the Verdict duchy (22)

**Corvane, the walled city**

| # | Place | | What it is |
|---|---|---|---|
| 1 | The Verdict Ring ★ | ⬛ | The colosseum. Law is made here. **Start** |
| 2 | The Sundial Market ★ | ⬛ | Stalls under a bronze sundial. Nara's ground |
| 3 | Hollowforge ★ | ⬛ | The blacksmith who arms both sides |
| 4 | The Chainhouse ★ | ⬛ | Where champions are kept between fights |
| 5 | Marblegate ★ | ⬛ | The great gate. Nothing leaves unread |
| 6 | The Sable Court | ⬛ | The ducal court. No weapons, worse dangers |
| 7 | The Iron Fane | ⬛ | The temple that certifies Verdicts. The fraud's heart |
| 8 | The Drowned Stair | ⬛ | Flooded undercity. The route Marblegate can't search |
| 9 | The Tally House | ⬛ | Every writ ever filed, and the clerk who files them |
| 10 | Lamplighters' Row | ⬛ | The poor quarter. Where losers' families end up |
| 11 | The Physicker's Walk | ○ | Where the Ring's survivors are patched or aren't |

**The uplands**

| # | Place | | What it is |
|---|---|---|---|
| 12 | Ravensgate Bridge | ⬛ | Toll bridge Thornhollow taxes |
| 13 | Thornhollow | ⬛ | The camp. A village in exile with a court of its own |
| 14 | Greywither Abbey | ⬛ | Ruined. Its cellar holds the Verdict rolls |
| 15 | The Weeping Quarry | ⬛ | The Ring's marble, cut by people sentenced there |
| 16 | Harrowfield | ○ | The farm village a rigged Verdict emptied |
| 17 | Coldwatch Tower | ○ | Watches the Vaskir road. Undermanned |

**The low country**

| # | Place | | What it is |
|---|---|---|---|
| 18 | Saltmoor | ⬛ | Fishing village. Takes anyone, remembers everything |
| 19 | The Ferryman's Post | ○ | Carries every faction, judges none |
| 20 | Duskbarrow | ⬛ | Barrows of the kings who ruled before the Ring |
| 21 | Blackreach Hold | ⬛ | The keep of whoever signed the writ |
| 22 | The Salt Kilns | ○ | Works the marsh. Owned by Serevane money |

### Vaskir, the Cold Marches — oath-law (14)

| # | Place | | What it is |
|---|---|---|---|
| 23 | The Rimegate | ⬛ | The mountain pass. Your first Vaskir ground |
| 24 | Hjarnmoot | ⬛ | The oath circle. Their Ring — and nobody fights in it |
| 25 | Vaskhold | ⬛ | Seat of the Thane-moot |
| 26 | Skarrow Longhall | ⬛ | A thane's hall. Hospitality is law here |
| 27 | The Bone Orchard | ⬛ | Where oathbreakers hang. Also where they are mourned |
| 28 | Fetterstone | ⬛ | The prison that holds people Aldermere wants back |
| 29 | The Silent Chapel | ○ | An older faith that predates all three legal systems |
| 30 | Grimsdal Mine | ○ | Iron. Vaskir sells it to Hollowforge |
| 31 | Elgmere Lake | ○ | Frozen half the year. Crossed at risk |
| 32 | The Whalebone Wharf | ○ | Where Serevane ships dock and are watched |
| 33 | Wolfsback Ridge | ○ | The high route. Impassable most of the year |
| 34 | The Drowned Fleet | ○ | Ships iced in a century ago and never freed |
| 35 | Wintermoor Crossing | ○ | The ford everyone actually uses |
| 36 | The Thane's Barrow | ○ | Where oaths are sworn over the dead |

### Serevane, the Thousand Coins — law by purchase (14)

| # | Place | | What it is |
|---|---|---|---|
| 37 | The Hundred Docks | ⬛ | Landfall. Everything arrives and is priced |
| 38 | The Gilt Exchange | ⬛ | Where judgments are listed and traded |
| 39 | Casa Verrine | ⬛ | The banking house that bought the succession |
| 40 | The Coinlord's Loggia | ⬛ | Serevane's government, such as it is |
| 41 | The Debtors' Terrace | ⬛ | Where people who lost are displayed |
| 42 | The Whisper Baths | ⬛ | Where the real deals are made, unclothed and unarmed |
| 43 | The Spice Vaults | ○ | Bonded warehouses. Also a smuggling route |
| 44 | The Lantern Quarter | ○ | Pleasure district and information market |
| 45 | The Glass Foundry | ○ | Serevane's other export |
| 46 | The Salt Bourse | ○ | Sets the price of the Aldermere marsh |
| 47 | The Merchant's Span | ○ | The bridge the whole republic crosses |
| 48 | The Anchorage Fort | ○ | Guns pointed at their own harbour |
| 49 | Cheira Isle | ○ | Quarantine island. Nobody checks who is on it |
| 50 | The Drowned Quarter | ⬛ | The old city, sunk. Nobody's jurisdiction |

**26 enterable, 24 map-presence.**

---

## 4. The forty characters

★ = art exists. ◆ = bespoke portrait needed. ○ = recurring but framed; earns a
codex card from play without bespoke art.

### Aldermere (16)

| Character | | Role |
|---|---|---|
| Cassian Vale | ★ | Champion of the Ring, ordered to lose the succession. Romanceable |
| Nara Voss | ★ | Writ-broker. Sells judgments. Romanceable |
| Duke Aldric Morrow | ◆ | Dying without an agreed heir. Sharper than he lets on |
| Lady Sereth Morrow | ◆ | His niece. Wants the Verdict fought honestly and will not say why |
| Lord Barran Coyle | ◆ | Wants it fought dishonestly. Signed something |
| Fane-Arbiter Vell | ◆ | Certifies Verdicts. Has certified one he knows was pre-written |
| Master Hob Cardew | ◆ | The Hollowforge smith. Arms the Ring and Thornhollow both |
| Wren | ◆ | The Tally House clerk. Has read everything. Terrified and useful |
| Gaunt | ◆ | Chainhouse keeper. Owns Cassian's contract |
| Iselle of Harrowfield | ◆ | Thornhollow's speaker. Lost her land to a rigged Verdict |
| Ord Blackreach | ◆ | Holds Blackreach. The name on the writ, or a decoy |
| Sister Mave | ◆ | Last of Greywither. Guards the Verdict rolls |
| The Ferryman | ◆ | Carries everyone, judges none, remembers all of it |
| Pell | ○ | Lamplighters' Row. A child who sees every arrival |
| Doctor Ansel | ○ | Physicker. Knows exactly which wounds were meant to kill |
| Overseer Krane | ○ | The Weeping Quarry. Sentences are labour here |

### Vaskir (10)

| Character | | Role |
|---|---|---|
| Thane Hjalvar Skarrow | ◆ | Hosts you. Hospitality is binding law and he will use that |
| Oathspeaker Ranneke | ◆ | Runs the moot. Can void an Aldermere Verdict — at a price |
| Brynn Skarrow | ◆ | Hjalvar's heir. Wants Vaskir to intervene. Romanceable |
| Fetterstone Warden Ulk | ◆ | Holds a prisoner Aldermere wants very badly |
| Torv the Unsworn | ◆ | Refused an oath and lives outside all law. Guide, if paid |
| Grandmother Aske | ◆ | The Silent Chapel. Older faith, longer memory |
| Kestrel of the Ridge | ○ | Smuggles across Wolfsback |
| Hedda Grimsdal | ○ | Runs the mine. Sells iron to both sides |
| The Bone Orchard Keeper | ○ | Buries oathbreakers and remembers their names |
| Sig | ○ | A boy who has crossed the pass eleven times |

### Serevane (10)

| Character | | Role |
|---|---|---|
| Coinlord Aurel Verrine | ◆ | Bought the succession. Does not consider this a crime |
| Ysolde Verrine | ◆ | His daughter and auditor. Found the discrepancy herself |
| Factor Meno | ◆ | Runs the Gilt Exchange floor. Will list anything |
| Captain Sabra Idre | ◆ | The Hundred Docks. Smuggles for the side that pays. Romanceable |
| The Anonym | ◆ | Whisper Baths broker. Nobody has seen their face |
| Salvat | ◆ | Debtors' Terrace. Was a Coinlord. Still trades from the railing |
| Bellara | ○ | Lantern Quarter. Sells what people said, not what they did |
| Harbourmaster Onn | ○ | Logs every hull. Has logged the wrong ones |
| Vito of Cheira | ○ | Quarantine island. Nobody checks his manifests |
| The Drowned Quarter Boy | ○ | Lives where no law reaches. Knows the tunnels |

### Wandering (4)

| Character | | Role |
|---|---|---|
| The Herald | ◆ | Announces Verdicts in all three realms. Neutral, and knows everything |
| Marek Sain | ◆ | A champion who lost and lived. The player's dark mirror. Romanceable |
| Old Coll | ○ | Travelling rumour-monger. The player's cheapest information |
| The Second | ○ | Whoever stands beside you at a Verdict. Changes by save |

**24 bespoke portraits (22 new), 16 emergent.** Five romanceable across three
realms, all with a real reason to refuse — see bible §11.

---

## 5. Map art vs location art — they are different images

Yes, and deliberately so. Every place has up to two pictures and they share
almost nothing.

| | **Map sprite** | **Location scene** |
|---|---|---|
| Camera | Three-quarter overhead, ~45° | Ground level, standing in it |
| Aspect | Square | Portrait 9:16 |
| Background | Flat chroma green, cut out | Full bleed |
| Size on screen | 60–200 px | Entire phone |
| Job | A **readable silhouette** at thumbnail size | An **empty stage** for character cutouts |
| Who has one | All 50 | The 26 enterable |

Two different pictures of the same building, made for two different jobs. A
scene shrunk to map size is illegible mush; a sprite blown up full-screen is a
blurry model of a building rather than a place you are standing in.

### The anchor-motif rule

The one thing that must survive between them is a **single distinguishing
feature, present in both images**, so the player recognises the place they
tapped.

| Place | Anchor motif |
|---|---|
| The Verdict Ring | Crimson-and-indigo banners between travertine arches |
| The Sundial Market | The verdigris bronze sundial on its stepped pedestal |
| Hollowforge | The tall soot-blackened chimney and its glowing forge mouth |
| The Chainhouse | The row of small barred windows over a sand yard |
| Marblegate | Twin drum towers and the raised iron portcullis |

Add one line to each future prompt pair naming the motif. **The five already in
flight need no change** — each motif is already written into both its sprite
and its scene prompt.

---

## 6. How this will fare — the honest read

**Art risk: manageable.** The real threat at 88 assets is style drift across
batches. Two controls: generate strictly by family (all plates, then all
archetype sprites, then all scenes) so drift shows up as a visible break rather
than a slow slide; and I run a normalisation pass in `sharp` — white balance and
levels matched to one reference plate — before anything is placed.

**Content risk: this is the real one.** Fifty places is four times the
*writing*, not four times the art. The production plan's own warning applies:
four excellent places beat a hundred thin ones. So the density stays
concentrated:

```
Chapter I    Aldermere only, 8 places open, everything else fogged
Chapter II   the Vaskir road; the writ leaves the duchy
Chapter III  Serevane; who paid, and the succession Verdict
Post-reign   Petitions across all 50, generated inside authored frames
```

Fifty places is a **map that is fifty places from turn one** and a *story* that
opens eight of them. That is the difference between an open world and a thin
one, and it is entirely a pacing decision, not an art decision.

**Where the expansion genuinely pays:** the archetype library and the three-realm
frame are the template. An isekai pack reuses the same 16 archetypes as guild
halls and floating markets, the same three-legal-systems spine as three magic
orders, and the same 26/24 enterable split. The second world costs a fraction
of the first, which is the actual claim we are making.
