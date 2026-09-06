# The Iron Verdict — World Bible

Replaces **Ashen Circuit** entirely. World key `iron-verdict`, seeded fresh at
version 1; the `ashen-circuit` definition and every one of its S3 objects are
retired once the new art is published.

> Why the rename: the old names — Cinder, Ash, Ember — were doing the art
> direction. Every prompt that carried them produced volcanic rock and glowing
> orange ground, which is why the map read as a dragon's den. The fix is not to
> hide the names from the prompts. It is to pick names that seed the *right*
> picture. Every place name below is chosen so that feeding it to an image
> model pulls toward stone, iron, water, timber and thorn.

---

## 1. The premise

In the duchy of **Aldermere**, no great dispute is settled by a judge. It is
settled in the **Verdict Ring** — a Romanesque arena in the capital city of
**Corvane** — by champions fighting on behalf of the claimants. A Verdict is
law. It cannot be appealed. It is carved into the Ring's inner wall and that is
the end of the matter.

Which means: *whoever owns the champions owns the law.*

The Duke is dying without an heir the court agrees on. The succession will be
decided by a Verdict. **Cassian Vale**, the reigning champion of Corvane, has
been quietly instructed to lose it.

The player arrives in Corvane as a nobody on the day of a Verdict, and finds
the sealed writ that proves the outcome was signed **before** the fight.

### The political spine

Three factions want the writ, and each is right about something:

| Faction | Wants | Truth they hold |
|---|---|---|
| **The Sable Court** (the nobles) | The succession to stay quiet | A public scandal means open war between houses |
| **The Iron Fane** (the temple that certifies Verdicts) | The writ destroyed | If Verdicts are known to be bought, every judgment for a century is void |
| **Thornhollow** (the "bandits") | The writ made public | They are not bandits. They are farmers whose land was taken by a rigged Verdict nine years ago |

The bandits being *dispossessed plaintiffs* is the load-bearing idea. It turns
the countryside from a place with random enemies into a place with an argument.

### The characters we keep

Both existing portraits stay and are **not** regenerated.

- **Cassian Vale** — champion of the Verdict Ring. Bronze lorica, crimson
  tunic, grey ragged cloak. Ordered to throw the succession Verdict. Knows he
  is being used; does not yet know by whom.
- **Nara Voss** — writ-broker of the Sundial Market. Blue travelling coat,
  lantern, sealed papers, satchel of keys. Buys and sells sealed judgments.
  She can read the writ. She will also sell you out if the price clears.

### Kin — who is related to whom

Canon as of Sept 6 2026. These were **not** in the original bible; they were
inferred from surnames while the cast was being written, and are recorded here
rather than left in a data file so that the next person to author against this
world reads the same facts.

The inference is sound because in this world a house's name is on its building:
Casa Verrine and Skarrow Longhall are named for the families that hold them.

- **Lady Sereth Morrow is Duke Aldric Morrow's niece.** This is the one with
  teeth. It does not merely place her at court — it makes her *a claimant*, and
  she wants the succession settled on herself. The political spine in §1 lists
  three factions who want the writ; Sereth is a fourth interest, a person who
  wants the throne, and she is inside the Court rather than opposed to it.
- **Ysolde Verrine is Aurel Verrine's niece**, and she audits her own house's
  books. She holds the countersignature on the succession writ and whose hand
  it is — the answer to the central question. Placing it with her makes the
  reveal a betrayal of family rather than of an employer.
- **Brynn Skarrow is Hjalvar Skarrow's daughter**, and her father intends to
  trade her oath for a treaty. That is what gives Skarrow Longhall a reason to
  matter beyond its hospitality law: there is someone in it who wants out.

### What the Duke does and does not know

Duke Aldric let the Iron Fane certify judgements he knew to be false. This is
**guarded knowledge**, gated behind `writ_read` — he will not volunteer it, and
it is not available in his first scene.

The distinction is mechanical, not stylistic. A character's open knowledge is
freely revealable in conversation, so a fact placed there can surface as small
talk. His complicity is the story's answer, and an answer that arrives before
the question has been asked is not a reveal. Note also that it reframes an
ending: *The Bloodless Crown* means something very different once the head of
the Court is known to have permitted the fraud.

---

## 2. Places

⚠️ **This section was rewritten on Sept 6 2026 from the live data, which had
outgrown it.** The previous version described sixteen locations across three
plates and named two places — Saltmoor and Duskbarrow — that were never built.
The world as shipped is **thirty locations across five plates and three realms**,
and it gained two realms the bible had never mentioned: Vaskir in the north and
Serevane across the water. Anyone reading the old section would have authored
against a map that does not exist; one agent writing post-ending content hit
exactly that and had to work around it.

Generated from `everlore-server/src/worlds/data/iron-verdict.json`. If the two
ever disagree again, **the data is right** — it is what the renderer draws and
what the plates were painted for.

Names stay deliberately architectural, not elemental.

### PLATE VASKIR — The Cold Marches

| Place | What it is | Starts |
|---|---|---|
| **Grimsdal Mine** | Iron. Sold south without much asking. | rumoured |
| **Vaskhold** | Seat of the Thane-moot. | rumoured |
| **Hjarnmoot** | The oath circle. Nobody fights in it. | rumoured |
| **Fetterstone** | Holds prisoners Aldermere wants returned. | rumoured |
| **The Bone Orchard** | Where oathbreakers hang, and are mourned. | rumoured |
| **Skarrow Longhall** | Hospitality is law under this roof. | rumoured |
| **The Rimegate** | The pass. Everything north begins here. | rumoured |

### PLATE NORTH — Aldermere — the uplands

| Place | What it is | Starts |
|---|---|---|
| **Coldwatch Tower** | Watches the northern road. Undermanned. | rumoured |
| **The Weeping Quarry** | The Ring’s marble, cut by the sentenced. | rumoured |
| **Thornhollow** | Not bandits. Plaintiffs, with a court of their own. | rumoured |
| **Harrowfield** | A village a rigged Verdict emptied. | rumoured |
| **Ravensgate Bridge** | A toll the Court pretends not to know about. | sealed |
| **Greywither Abbey** | Its cellar holds the Verdict rolls. | rumoured |

### PLATE CENTRAL — Aldermere — Corvane

| Place | What it is | Starts |
|---|---|---|
| **The Sable Court** | No weapons. Worse dangers. | rumoured |
| **The Verdict Ring** | Law is made here, and it cannot be appealed. | open |
| **The Chainhouse** | Cassian is not a prisoner, and cannot leave. | sealed |
| **The Iron Fane** | It certifies every Verdict. It certified this one. | rumoured |
| **The Sundial Market** | A new lead waits among the lantern stalls. | sealed |
| **Hollowforge** | The smith arms the Ring, and quietly arms its enemies. | sealed |
| **The Drowned Stair** | The one road out that cannot be searched. | rumoured |
| **Marblegate** | Nothing leaves the city unread. | sealed |

### PLATE SOUTH — Aldermere — the low country

| Place | What it is | Starts |
|---|---|---|
| **Blackreach Hold** | The keep of whoever signed the writ. | rumoured |
| **The Ferryman’s Post** | He carries every faction and judges none. | rumoured |

### PLATE SEREVANE — The Thousand Coins

| Place | What it is | Starts |
|---|---|---|
| **The Spice Vaults** | Bonded warehouses, and a way under them. | rumoured |
| **Casa Verrine** | The house that bought the succession. | rumoured |
| **The Debtors’ Terrace** | Where people who lost are displayed. | rumoured |
| **The Gilt Exchange** | Judgements are listed here like cargo. | rumoured |
| **The Whisper Baths** | The real deals, unclothed and unarmed. | rumoured |
| **The Hundred Docks** | Everything arrives here, and everything is priced. | rumoured |
| **Cheira Isle** | Quarantine. Nobody checks who is on it. | rumoured |

Eleven of these start `rumoured` and are not drawn at all until the fiction
names them, which is what makes the map grow during play rather than being laid
out on turn one. See section 3.

---

## 3. How this feels open

Three visibility tiers, not two. The third is the whole trick.

```
open      travellable now; drawn in full colour
sealed    drawn on the map with a lock treatment. You can SEE it and you are
          told WHY it is shut. This is a promise, not a wall.
rumoured  NOT ON THE MAP AT ALL. The region is under fog. It is revealed —
          with a discovery beat — the moment the fiction first names it.
```

A `rumoured` place costs no bespoke art until it becomes playable: fog lifts to
reveal the shared **sealed marker** sprite, and only a location you can actually
enter needs its own landmark and scene.

So the map *grows while you play*. That is what sells "open world" far more
cheaply than raw square footage, and it is why eleven of sixteen places start
hidden.

**No route lines are drawn.** Adjacency stays in the data as travel rules; the
visible route is the road painted into the terrain plate. The old connecting
lines are deleted.

### Chapter I unlock graph

```
The Verdict Ring  (start, open)
  │ witness a Verdict
  ├─▶ The Sundial Market            [open]
  │     │ Nara reads the writ
  │     └─▶ rumour: The Drowned Stair, Thornhollow   [fog lifts → sealed]
  │ Cassian's trust
  ├─▶ The Chainhouse                [sealed → open]
  │     └─▶ Hollowforge             [sealed → open]
  │ carry the writ
  └─▶ Marblegate                    [sealed → open]
        │ leave the city
        └─▶ Ravensgate Bridge       [open]
              │ spare the toll-takers
              └─▶ Thornhollow       [sealed → open]

  everything else remains rumoured → Chapter II
```

Flags: `verdict_witnessed`, `cassian_trust`, `writ_read`, `gate_passed`,
`thornhollow_pact`, `stair_open`.

### The Chapter I quest — *The Bloodless Crown*

1. Witness a Verdict in the Ring. The loser is carried down the Drowned Stair.
2. Cassian shows you the Chainhouse. He is not a prisoner and cannot leave.
3. Nara reads the sealed writ: the succession Verdict is dated **before** the
   fight it decides.
4. Get the writ out of Corvane — bribe, forge, fight, or take the Stair.
5. Thornhollow's court recognises the seal. It is the same one that took their
   land. Ally with them, sell them, or walk.

Beat 5 sets the Chapter II opening and decides which of the eleven hidden
places surfaces first.

---

## 4. Combat — the Verdict duel

Verdicts are the fight system, and they use the **King's Choice** presentation
the user specified. This is a *scene skin*, not a new engine, and it needs no
character art beyond what we already have.

```
full-bleed arena background (portrait, low camera on the sand, packed crowd)
  opponent cutout      upper right, ~55% frame height
  player/champion cutout   lower left, ~65% frame height, overlapping the rail
  opponent bar         top:  name, level, attack, long HP bar
  player bar           bottom: portrait roundel, attack, long HP bar
  floating damage      large numerals rising and fading over the struck figure
  critical banner      wide red display text across the midline
  bark bubble          parchment speech bubble beside the speaker, 1 line
  skip control         bottom right
```

Rules that keep it honest:

- The duel is an **authored state machine**, same as every other world action.
  Damage numbers are theatre over a server-resolved outcome; the client never
  decides who wins.
- Every duel is a *Verdict* — it settles a named dispute and writes a canon
  consequence. There is no grinding, and no fight without a stake.
- Barks come from the character's own voice and context packet, so a duel is a
  scene with dialogue, not a damage screen.

## 5. Travel — walking POV

Moving between places plays a short first-person walking transition rather than
a cut. Pure code, no assets:

- destination background enters at ~1.08 scale and eases to 1.0 while drifting
  toward the location's anchor;
- a small sinusoidal vertical bob (~6 px, ~2.2 Hz) with a coupled sub-degree
  roll — the *shake* — damped to zero on arrival;
- vignette breathes in and out once; footstep audio and light haptics carry the
  weight;
- total ≈ 900 ms, and it is skippable.

Long routes (leaving the city, crossing the bridge) run the loop longer and
add one wipe of foliage or dust so distance is felt.

---

## 6. Why this is a template, not one world

The moat is that this whole structure is genre-swappable. Nothing in the
renderer knows the word "medieval".

```
GenrePack
  terrain plates      N square plates stacked into one tall map
  landmark sprites    matted art + alpha hit geometry + state variants
  scene backgrounds   one per playable location
  shared kit          fog texture, sealed marker, scatter props
  style bible         one paragraph, prepended to every art prompt
  vocabulary          what a "Verdict", a "route", a "faction" is called
```

`iron-verdict` is the **medieval** pack. An isekai pack swaps the plates for
floating islands and guild halls, the Verdict for a guild rank duel, and the
sealed marker for a barrier rune — and every line of map, travel, unlock, fog,
duel and scene code is unchanged. The style bible is the only thing that has to
be rewritten per pack, and it is one paragraph.

That is the product claim: **we can author a new open world by delivering an
art kit and a state machine, not by writing a new game.**

### The socket rule — carry this into every future plate prompt

> **Superseded in direction, not in fact.** Everything in this section stays true
> for the sprites that shipped. The September 2026 decision is to generate future
> landmarks as 3D models rendered through a *declared* camera, which removes the
> camera-matching problem this section works around. See
> [`INTERACTIVE_MAP_PIPELINE.md`](INTERACTIVE_MAP_PIPELINE.md).


A terrain plate is not a finished picture. It must paint the world **around**
the landmarks and leave the landmarks out, because the landmark sprites are
composited on top of it and are the only interactive things on the map.

The Iron Verdict plates got this right by accident of good prompting: Corvane
came back with four empty paved plazas and a gap in the wall on the main road,
and those holes are exactly where the arena, the court, the market, the
chainhouse and the gate now sit. Nothing had to be repainted.

It cannot be left to accident again. Every plate prompt must say:

> Leave several **empty paved plazas and courtyards** of varying size
> distributed across the settled area, and a **gap in any city wall where a
> main road crosses it**. These are deliberate empty sockets — do not fill them
> with buildings, monuments, statues, fountains, tents, crowds or market
> stalls. Bare ground, correctly lit and textured, is the wanted result.

A plate that paints its own cathedral in the only open square forces every
landmark onto a rooftop, and there is no post-process that fixes it.

Two consequences follow for placement:

- **Sprites are drawn at 0.16 of map width** (times the authored per-location
  scale). This is set by the plate's own building scale, not chosen for looks.
  At larger values a landmark towers over its own city and reads as pasted on.
- **Coordinates are measured against the delivered plate, never authored
  blind.** The sockets are wherever the generator put them.

#### Seating a sprite that must join painted geometry

A landmark that stands in its own plaza only needs its socket. A landmark that
must *continue* something already painted — a gate in a wall, a bridge between
two banks — has to be solved by measurement, because judging it by eye fails
in a specific, repeatable way: the sprite is drawn at a plausible-looking size
and ends up floating in front of the wall rather than closing it.

Marblegate took four regenerations before this was written down. What finally
worked, in order:

1. **Overlay a plate-coordinate grid on the gap** and read the two wall ends
   off it. Corvane's gap runs x 520 → 690 at 1122px plate width — 170px, and
   its centre, 605, is the sprite's x. Impressions of this gap were wrong by
   40px in both directions on separate attempts.
2. **Overlay the same grid on the matted sprite** and read where its own
   landmarks sit as fractions of its box. Marblegate's towers span the full
   width, so tower-to-tower span = draw width, and `scale` = 170 / (1122 ×
   0.16) = 0.95.
3. **Set y from the ground line, not the silhouette.** The tower bases sit at
   .68 and .81 of the sprite's height; matching those to the wall's own base
   line on each side gave the same centre y within 5px. That agreement is the
   check — if the two sides disagree, the sprite's camera does not match the
   plate's and no placement will save it.
4. **Composite offline and view at 3×** before publishing. Every seating fault
   found this session was invisible at 1× and obvious at 3×.

The sprite's camera must match the plate's *before* any of this is worth
doing. The reference that finally produced a usable gate was a native-
resolution crop of the gatehouse **as painted into the map**, handed to the
generator as an image to match — not a written description of an angle. Four
written descriptions of the angle failed; the crop worked first time.

---

## 7. Art direction, corrected

The previous prompt sheet asked for *flat hand-illustrated storybook* art. That
was wrong and would never have composited with the characters we already have.

Cassian Vale and Nara Voss are **detailed semi-realistic digital painting**:
cinematic warm key light, deep shadow, brass and crimson, visible material
texture. Everything new must match *them*, because they are the assets we are
keeping.

Architecture is **late-Roman into early-medieval** — a Romanesque arena of pale
travertine arcades, hung with heraldic banners, inside a walled city of timber
and slate. That is exactly the register of the reference screenshots, and it is
what a colosseum with a crowd actually needs.

Prompts live in `IRON_VERDICT_ASSET_PROMPTS.md`.

---

## 8. The end goal

The campaign question is one sentence: **who wears the crown of Aldermere, and
does the Ring survive?** The writ is the object that decides it, and the whole
map is arranged around getting it, reading it, moving it, and spending it.

Four endings. Each is reachable, none is the "good" one, and each one is the
direct arithmetic of who you allied with and what you did with the writ.

| Ending | How you get it | What it costs |
|---|---|---|
| **The Verdict Upheld** | Sell or burn the writ | The fix stands and you are paid for it. Cassian throws the succession Verdict and dies on the sand |
| **The Bloodless Crown** | Take the writ to the Sable Court | Succession passes with no fight and the Ring is reformed. Thornhollow get nothing, and know it |
| **The Broken Ring** | Give the writ to Thornhollow | A century of Verdicts is void. Land goes back. So does open war between the houses |
| **The Iron Verdict** | Destroy the writ and fight the succession Verdict yourself, honestly | The system stands, and it is now yours to abuse |

The endgame trigger is the **succession Verdict**: an authored date on the
world calendar. It arrives whether or not you are ready. That deadline is what
turns exploration into pressure — every place you have not opened yet is time
you are spending.

### Progression the player can see

Three tracks, all already backed by systems we have:

- **Marks** — chronicle achievements. Awarded for consequence, never for
  grinding: *first Verdict witnessed*, *a champion's trust*, *the writ read*,
  *walked out of Corvane unsearched*, *seven places found*, *a house made an
  enemy*, and one hidden Mark per ending.
- **Standing** — a value per faction (Court, Fane, Thornhollow, the Ring
  itself). Standing gates dialogue, prices, and which of the eleven hidden
  places surfaces first. Raising one lowers another; there is no route to all
  four.
- **The Ledger** — a running list of every Verdict you have witnessed or
  fought, with what it decided. It is the player's own evidence file, and in
  three of the four endings it is what they hand over.

## 9. What actually produces the open-world feeling

Not square footage. Six mechanics, in order of how much they buy:

1. **A map that grows.** Fog lifts and new markers appear *while you are
   playing*, because the fiction named a place. Nothing else creates "there is
   more out there" as cheaply.
2. **Legible locks.** A sealed place tells you what it wants. A wall you
   understand is a goal; a wall you do not is a dead end.
3. **More ways out than in.** Marblegate can be bribed, forged, fought, or
   skipped entirely via the Drowned Stair. Every gate in Chapter I has at least
   two solutions and they cost different things.
4. **Places that change behind you.** Standing, time of day, and quest stage
   swap sprite variants and repopulate scenes. Returning is never a re-read.
5. **Consequence that travels.** What you do in the Ring is known in the Market
   before you get there. This is what our memory system is *for*, and it is the
   thing a hand-authored game cannot cheaply do.
6. **A clock.** The succession Verdict is coming. Freedom only reads as freedom
   when spending it costs something.

---

## 10. After the ending — The Reign

The ending is not a credits roll. It is a **rule change**. The world stays
loaded, the player stays alive unless they chose otherwise, and every place
they opened stays open — but the map now runs under different law, because
each ending puts the player in a different relationship to power.

| Ending | The verb it gives you |
|---|---|
| **The Verdict Upheld** | *Spend.* You are rich and complicit. The Court owns your calendar and sends for you |
| **The Bloodless Crown** | *Reform.* The Ring answers to a new charter you helped write, and every house you embarrassed remembers |
| **The Broken Ring** | *Hold.* Aldermere is contested ground. Places change hands. Roads close |
| **The Iron Verdict** | *Summon.* You own the Ring. You schedule Verdicts — which means you decide what the world argues about next |

That last one is the interesting case: the player becomes an author of world
events, using the same authored machinery the game used on them.

### Petitions — where content comes from once the script runs out

Standing makes people bring you things. A **Petition** is a small dispute — a
boundary, a debt, a killing, an inheritance, an accusation — with named
parties, a real stake, and two or more resolutions. The player judges it,
fights it as a Verdict, refuses it, or sells it.

The frame is authored. The contents are generated. That split is the whole
design:

```
authored, never generated   places, routes, factions, canon facts, ending
                            states, what a Verdict legally IS
generated, always fresh     who is petitioning, what happened, what they want,
                            what they will admit, who is lying, the barks
```

Every resolved Petition writes to the Ledger, moves Standing, and can promote a
recurring petitioner into a full character with their own bonds. So the cast
grows out of play rather than out of a content pipeline.

### The honest limit

Our map silhouette is finite, because a place a player can *stand in* needs
art. The life inside it is not.

A generated Petition can name a new room, cellar, farmstead, or camp and make
it a real, remembered location — reusing the nearest kindred scene as its
backdrop and appearing on the map under the shared sealed-marker sprite. It
becomes a place with a name, a history, and people. What it does not get is its
own painting.

We should say that plainly rather than pretend the map is infinite. "Sixteen
painted places, unbounded life inside them" is a true claim and a good one.

### Death

The player can lose. Losing a Chapter I duel does not kill you — a **judgment
is written against you**, which is worse and more interesting: you carry it,
and everyone can read it.

Death becomes possible in the succession Verdict and afterwards. It is not a
game-over screen. The Ledger passes to someone who was standing near you, the
save continues from their side, and your own last Verdict becomes a fact other
people cite. This reuses the existing two-way death model rather than inventing
a mortality system.

---

## 11. Bonds and romance

Romance is **not a separate system**. Every character already carries bond
meters — affection, trust, rivalry, fear. Romance is what affection crossing a
threshold *unlocks*, gated by one authored beat so it cannot happen by
accident or by grinding.

### The two authored romances

Both have a real reason to refuse, which is the only thing that makes a yes
worth anything.

- **Cassian Vale.** He is owned. He believes he is going to die on a date
  already chosen for him, and will not start something he cannot finish.
  Reaching him means changing that fact, not saying the right line. In *The
  Verdict Upheld* he dies — so a romance turns the mercenary ending into the
  cruel one, and it should.
- **Nara Voss.** She trades in people, likes you, and knows those two things
  are in conflict. Her romance is not "does she care" — it is whether she stops
  selling. She can betray you *after* the romance lands, and it still counts.

Romance changes the epilogue, the Petitions you are offered, and who speaks for
you at the succession Verdict.

### Player-created love interests

Yes — and this is close to free for us, because it emerges from systems that
already exist.

A character the player invents through play already earns a card and bond
meters once they recur. Romance rides the same rails: keep meeting someone,
affection accrues, the same authored gate applies, and the relationship becomes
canon — remembered, referenced by others, carried into the epilogue.

Rules that keep it honest:

1. **Consent is modelled, not decorative.** Generated characters have their own
   standing, loyalties and refusals. Affection is earned against a real
   position, never assumed by proximity.
2. **The existing content bounds apply unchanged** — the soft in-fiction
   boundaries and the hard input guard. Romance does not get an exemption and
   no bypass gets built for it.
3. **A romance is a canon fact.** It goes in the codex, other characters know,
   and it cannot be quietly forgotten on a later turn. A relationship the world
   fails to remember is worse than none.

---

## 12. Glory — the ambition ladder

"I achieved the goal, now what" needs an answer with rungs, not a sandbox.

```
Champion    win Verdicts. Standing with the Ring
Patron      own a champion's contract. Others fight your Verdicts
Landholder  hold a place. Its sprite changes state and it produces and it
            can be taken from you
Claimant    a house backs your name. The Court must answer you
Duke        the last Verdict. It is a real fight and it can be lost
```

Each rung is a handful of Petitions plus one Verdict that cannot be avoided.
Standing decides which rungs are even offered — a player who broke the Ring
cannot become its Patron, and should be told why.

This is deliberately the same shape as Chapter I: pressure, a legible lock, a
date, a fight, a consequence written down. The player has learned these rules
by the time the authored story ends, which is exactly when we hand them over.
