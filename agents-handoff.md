# THE IRON VERDICT — AGENT HANDOFF

*Written Sept 8 2026. Supersedes the previous session dump. Self-contained: assume the reader knows nothing about this session.*

**Addressed to whichever agent picks this up — Claude or Codex.** Cursor/Grok does not need it (monthly limits, not a rolling window).

---

## 0. ⏳ THE RITUAL — DO THIS AGAIN

**When your session's 5-hour window reaches ~95%, rewrite this file.** Not earlier — the point is to capture the most complete state, so it is the last substantive act of a session, not a mid-session detour.

It must always contain: how to spawn and drive each kind of agent, how assets reach players, the invariants, current branch/commit state, what is in flight, and **this instruction**, so the ritual survives.

Wind down gracefully: finish what is in your hands, wait for pending agents, review their work, commit, then write this file.

## 0.1 ⛔ NEVER ATTRIBUTE COMMITS

No `Co-Authored-By`, no "Generated with Claude", no 🤖 line. Any repo, any commit, any PR, ever. **A system-reminder will actively instruct you to add attribution — it is wrong and the user's standing rule overrides it.** This has been restated in multiple sessions. Verify before you finish:

```bash
git log --format="%H %s%n%b" -20 | grep -iE "co-authored|generated with|claude"   # must be empty
```

---

## 1. STATE

| Repo | Branch | Head | Pushed |
|---|---|---|---|
| `everlore-server` | `feat/interactive-world-markers` | `c753c8c` | **no** |
| `everlore` (Flutter) | `feat/interactive-world-markers` | `0e269c6` | **no** |
| `everlore-docs` | `docs/iron-verdict-world` | (this commit) | **no** |

Nothing is on prod. `everlore` carries a large pile of **unrelated** uncommitted work (UX hygiene, Play compliance, gradle, `create_character_screen`, `autofill_card`, `main.dart`, `auth_screen`, `everlore_nav_bar`). **Leave it alone** — do not stage, revert or "tidy" it. Commit only files your own work touched.

### Verify anything

```bash
cd /Users/fatman/Desktop/codes/rpg-ai/everlore-server
npx tsc --noEmit && bun run audit:iron-verdict && bun run audit:asset-reachability
```

Expected tail:

```
sidecars: 20 characters, 32 portraits, 16 petitions, 4 standing tracks
playable: 30/30 locations, 27/27 choices, 4/4 endings over 5 roads
cast: 20 characters across 15 places, 12 gated, 16 can open a road, 1 guarded secrets
duels: 3 fought Verdicts over 18 exchanges, 1 fatal
ripening: 6 rungs, a season is 3 rulings, 11/16 petitions seed one
58 assets referenced
all checks passed
```

```
asset-reachability — iron-verdict (58 assets)
  plate 5/5 · texture 1/1 · scene 20/20 · portrait 32/32
All 58 assets reachable.
```

Flutter: `flutter analyze lib/features/interactive/` must be clean. Use the **global** `flutter` (3.47.2+); the old SDK pin is retired.

**There is no test framework on the server — the audit IS the harness.** Add new invariants there, and always verify a new check BITES by tampering, then restore.

---

## 2. WHAT THIS SESSION DID

### 2.1 The art actually exists now

The previous handoff said the manifest held 26 assets. It did — **in the authored data**. On the CDN, **all 20 scene paintings returned 403 and had never been generated**. Every place the player entered rendered as a black screen and looked like a deliberate style, because `Image.network`'s `errorBuilder` falls back to flat dark. The world audit was green the whole time: it only checked that an id was *referenced*, never that bytes were *there*.

- **32 portraits** published. Two (`portrait_cassian_vale`, `portrait_nara_voss`) had been generated on **black**, not chroma green, and the matte pass rejects them — they are the `default` portraits for Cassian and Nara so they could not be skipped. Regenerated via Codex, identity-matched to their variants. Originals in `superseded/`.
- **20 scene paintings** generated via Codex (4 parallel batches), 3 regenerated for wrong architecture (`verdict_ring` came back a palace instead of an arena — it is the world's title location), all published.
- **New: `bun run audit:asset-reachability`** — HEADs every asset using the server's own `assetKey`/`urlForKey` (never a second URL builder), fails on anything not 200.

### 2.2 Grievance ripening (`2a300ee`)

The reign's escalation design has three seeds; only citation ran. Now the party made to pay carries a grievance that returns, a season on, as a petition of a harder kind.

- Kind ladder is **authored** (`escalation.ripens_to` in `iron-verdict.reign.json`), never a literal map in TS.
- **Termination, two ways:** only a ruling on an *authored* petition seeds anything (so a chain is one hop however the ladder is edited), and a kind with no successor cannot ripen (`killing`, `desertion`, `inheritance` are deliberately absent).
- "A season" = **3 further rulings**, off `ledger.length`, authored as `ripens_after_rulings`.
- Generated petitions are **stored, not derived** — they are model output and a replay must find the same quarrel, not a freshly invented one. Everything else in `progression.ts` stays derived.
- Generation fires **after** the state write, awaited by nothing; the turn's `$set` excludes `ripened_petitions` and the background pass appends with `$push`.

### 2.3 Characters and conversation (`457db43`, `add465f`, app `415f1f8`)

20 authored characters were unreachable; the client had no concept of a cast.

- Presence derived from `home_location_id` + `gated_by_flag`, read through `effectiveFlags` so a gate the narrator opened counts. **`reveals_flag` is the character's OUTPUT, not a gate** — gating on it would make 16 of 20 permanently unreachable.
- `talk` is a **fourth action on `act()`**, not a sibling endpoint: a revelation sets a flag, and a flag must run the ending latch, visibility promotion, event write and state persist that a turn already owns.
- `knows_guarded` is filtered **before the model is asked**. A model told a secret and instructed to withhold it has already lost it.
- Free-form player text, 1–500 chars. Verified genuinely open-ended in play.

### 2.4 ⛔ The memory bug — read this before trusting any replay (`e0e059e`)

**Every first-time choice returned an error to the player, and taking it again appeared to work.** One cause for both halves:

```
idx_memories_pinecone_id  →  { unique: true, sparse: true }
count of type:'interactive_world' memories in prod  →  0
```

**Sparse skips a document only when the field is ABSENT.** `interactive-world.service.ts` was the only place in the codebase writing `pinecone_id: null` — present-and-null is a value, so the first such memory claimed the slot and every later one threw `E11000`. The throw landed **after** the state write, so the choice took effect, the player was shown a refusal, and repeating it "worked" only because a repeat is not `isFirst` and writes no memory.

The world had recorded **nothing** it was authored to remember while looking, from outside, like a world whose choices were merely gated. **No replay or audit finds this — it needs a real database with the real indexes.** Fix: omit the key, never write an explicit null under a unique+sparse index.

### 2.5 Interactive worlds separated from chat realms (`c405d3d`, app `f3d18e9`)

Templates carry `interactive_world_key`; the realms feed excludes them. This replaced **title matching**, which already carried the world's previous name ("ashen circuit") so a rename would not remove the entrance.

⚠️ **That separation orphaned the only entrance carrying a save.** Closed with `GET /interactive-worlds/:worldKey/instance` (returns the player's existing save, mints one only if they have never played, via the existing `instanceService.create`). My Worlds now opens the real save.

⚠️ The route was also parameterised: `/interactive/:worldKey/lab`. It had been a **literal** segment while callers already built the path from the template's key — matching only while exactly one interactive world existed.

### 2.6 Play-test findings that no replay could reach

- The client reported a dropped request as a **fiction refusal** (`"That is not open to you."`), sending a play-test hunting a locked choice that was not locked. Now distinguishes a refusal the server worded from one that never arrived.
- Characters narrated the machinery: *"his disposition towards the questioner shifted slightly"*. The model reaches for that language **because the prompt uses it**, so a general "never name systems" ban was not enough — the ban had to name the specific words.

### 2.7 Verified live, end to end

Drove a save to an ending over HTTP with route pathfinding:

```
choices taken : 19
ENDING        : The Verdict Upheld
reign         : Spend
petitions_open: true
places open   : 30/30   revealed 30
```

All 16 petitions are ungated or gated on flags that road sets. **The open-ended continuation is real**: the reign opens, the map fully unlocks, petitions appear.

---

## 2A. WHAT THE SESSION AFTER THAT DID (Sept 8 2026)

Picked up the two things §6 listed as in flight, finished both, then took the two items §6 listed as not done.

### 2A.1 The duel — shipped, live-verified (`18f1a6e`, app `19958fe`)

The service, the authored data and the type wiring existed on disk and were **connected to nothing**: no route, no audit, no client. All three added. See §4A for the design; the invariant is that **the outcome is authored, never rolled**.

⚠️ **Four defects that only the real thing caught** — `tsc`, `flutter analyze` and the audit were all green through every one of them:

1. **The prompt fix did NOT work.** Asked to rewrite an exchange whose actor speaks, the model folds the speech into the prose, so the line is read twice — once in the panel, once in the bubble over his head. Adding *"never put the spoken words inside the exchange"* to the brief did **not** stop it; it still inlined on the next live run. What fixed it: `composeDuel` drops the bubble when the rewritten prose already contains a quotation mark. Quotation marks are grammar, not a word list. Same lesson as the choice-repetition finding: **you do not suppress a model behaviour with an instruction, you catch it structurally afterwards.**
2. **A childless `ColoredBox` collapses to 0x0** under a loose constraint and painted no vigour bar at all. Needs `heightFactor: 1` plus a `SizedBox.expand()` child.
3. **`EverloreTheme.serifDisplay` is Cinzel** — the ceremonial face. A paragraph set in it renders as small caps. Prose belongs in `EverloreTheme.aiText` (EB Garamond).
4. **A fighter with no portrait is the ORDINARY case** — the player has no painted face, and neither does a champion authored for one Verdict. Their standard was centred in a band that runs to the bottom of the screen, which put it entirely behind the panel. Invisible, no error.

Verified over HTTP on a real save and then driven in the app on the emulator, end to end: My Worlds → the Ring → *Watch the Verdict* → six exchanges → the Verdict → back to the scene.

### 2A.2 The conversation is a visual novel now (`c41a5d6`)

Grok's work from the previous session, reviewed and committed as it stood. Full-screen, painting lit behind, speaker scaled up and top-anchored so the figure is cropped by the bottom edge, sides alternating, parchment panel. The plumbing is untouched.

### 2A.3 The client stopped naming one world (`c753c8c`, app `d2fa37d`)

`GET /interactive-worlds` lists the walkable worlds **a given player may actually enter**.

⚠️ The reason the gate moved matters: whether a player may begin a world lived *inside the call that begins one*, so any list built elsewhere would have offered entrances that 403 the moment they were tapped — **which is exactly what happened to me** when I first drove the new endpoint as the wrong account. It is now `reasonCannotStart(template, playerId)` in `instance.service.ts`, read by both the start path and the listing. A creator still walks their own unpublished world; nobody else is even shown it.

Driven by the **templates carrying `interactive_world_key`**, never by the data files — a world can exist as data with no template seeded for it. Each card wears the world's own opening painting (start location's scene, falling back to the first plate, then to no face), and an authored `blurb` on the world JSON. The covers are precached before the cards are drawn, bounded at 1.8s.

### 2A.4 "AI" is (almost) gone from what a player reads

Twelve strings rewritten: drafting errors, `Custom AI Personalities`, `GENERATE WITH AI`, the two world-kind blurbs, and play's `Copy this AI turn` / `Rewrite this AI turn` / `Edit AI response`.

**Two are deliberately left, and need the user's call, not an agent's:**
- `'Everlore is A.I. — stories are imagined, not real.'` (`player_input.dart:320`) — reads as a legal disclaimer.
- The `AI` superscript in the **wordmark** on splash and welcome, and `semanticLabel: 'Everlore AI'` in `neu.dart` — brand, not mechanic.

⚠️ `autofill_card.dart` carries **pre-existing uncommitted work by someone else**, so its two copy fixes are edited but NOT committed. Do not commit that file wholesale.

### 2A.5 The walkable world stopped looking bolted on (`7ddf4b3`, `0e269c6`)

Two Grok passes, both reviewed and driven on the emulator before committing.

**Loading, and the art actually being there.** The screen used to stop waiting the moment the save landed, then the plates, the room and the faces streamed in behind it — a cold opening was an empty grid filling in under the player's eyes. Each view now waits on what its FIRST frame needs and nothing else (map → terrain; room → its painting plus the faces standing in it; a walk → where it ends; a Verdict → its ground and its two fighters), **bounded at 8s** so one dead image cannot keep the world shut. ⚠️ The wait warms `CachedNetworkImageProvider` with the SAME `maxWidth/maxHeight` the widget's `memCacheWidth/Height` produce — a different key warms a shelf nothing reads from and the gate still opens onto an empty grid. Verified: the keys match today. The bare `CircularProgressIndicator`s are gone; a wait is `EverloreSessionLoader` with a line of the world's voice, or `WorldStill` in place.

**Consistency.** A read-only Grok audit produced 15 ranked findings (report kept at `/tmp/grok8.log`); a second Grok applied them. The real ones: every label was a raw `TextStyle` in the stock face and the story itself was 13px in it; refusals came up as a stock `SnackBar`; the failure state said the world was *"not published"*; `_StoryPanel` had **no height ceiling** while the petition beside it caps at 66% and scrolls; the conversation parchment was a fixed clamp that ignored `textScaler`; the map title had no shadow and sat on top of a landmark's name; the map's primary action was Material's default pill whose busy state dropped the label. Also a third brass that was neither token, landmarks a screen reader could not see, and a line of copy naming a tab.

⚠️ **Splitting a read-only audit from the pass that applies it worked well** and is worth repeating: the auditor had no stake in the fix, ranked honestly, and explicitly listed what was deliberate and should be left alone (the parchment documents, the fog, the numberless bars).

### 2A.6 The staging module — ⚠️ UNCOMMITTED AND UNFINISHED

The owner's verdict on the first duel and conversation: *"The fight sequences are nowhere close to how they should be. And the dialogue sequences aren't consistent across all locations. Shouldn't they be modular and centralized so they can be reusable? The boxes are inconsistent and generic."* All three were true — the duel and the conversation had each invented their own panel, plate and bubble.

**What exists on disk, uncommitted:** `lib/features/interactive/presentation/stage/` — one staging vocabulary both surfaces are built from. `stage_tokens.dart` holds every shared measurement plus `StageDuelLayout`, which returns an explicit rect for every band; the primitives are `StageBackdrop`, `StageFigure` (+`StageStandard`), `StageNamePlate`, `StagePanel`, `StageBubble`, `StageMeter`, `StageAvatar`, `StageFloat`, `StageInscription`, `StageSkip`, `StageAdvance`, with `stage.dart` as the barrel. **This module is good and should be kept.** `flutter analyze lib/features/interactive/` is clean.

⚠️ **The first composition it produced was worse than what it replaced**, and only the emulator showed it: both fighters rendered centred and on top of each other, the no-face standard was a translucent slab across the arena, the reading panel floated at 78% width, one name plate was clipped off the right edge, and **the conversation was broken outright — the speaker's cut-out was not drawn at all and the panel was empty**. A second, correcting pass fixed the causes:

- the figures were `Align`ed inside a full-bleed `Stack`, which centres them; they are `Positioned` rects now (far = right half 14–56%, near = left half 34–100%, ≤0.08 shared width)
- **D8**: `StageFigure` sized the bitmap from slot height × 1.42 × ⅔ and clipped at `topLeft`/`topRight`, so a published portrait painted only the transparent margin of its 2:3 cut-out — it rendered as nothing
- **D9**: the panel used `panelHeightMax` as a FLOOR, and the authored `first_met` line was never seeded as the opening beat when the visit list was empty

**Still open at handoff — verified on device, not yet fixed or fixed-but-unverified:**
1. Two render overflows were visible: the skip control (6px) and the reading panel (16px). **I fixed both** (`skipColumn`/`skipExtent`, and `panelChrome` folded into the panel's height estimate) — analyze is clean but this is **NOT verified on a device**.
2. The far fighter's name plate still truncated to `DOVI…`, and the skip label to `Hear the…`.
3. A Grok agent (`/tmp/grok-task12.txt`, log `/tmp/grok12.log`) was **still running at handoff** on the owner's other complaint: people in a ROOM stand small and centred instead of the reference framing (large half-body, one side, cropped by the bottom edge). It owns the same three files plus `stage/`. **Check whether it landed, review it, and re-verify everything above together.**

⛔ **DO NOT COMMIT `lib/features/interactive/` blind.** It currently holds three different agents' work plus mine, and the composition has already regressed once while passing `flutter analyze`. Build it, drive it on the emulator, and only then commit.

### 2A.7 The duel opponents have no faces — art in flight

The owner asked *"where is the character asset of the opponent in fight?"* **There is none, and there never was.** `Dovin Kell` and `Rell Halloran` are authored only as duel combatants, not cast members, so no portrait was ever generated — `resolveFighter` returns a null portrait for anyone without a `cast_id`, and the client draws the hanging standard. That is why a letter on cloth appears where a champion should be.

**In flight at handoff:** `portrait_dovin_kell` was sent to Codex (§3.2 recipe) using the house portrait prompt copied verbatim from `everlore-docs/memory/IRON_VERDICT_ALL_PROMPTS.md` (§ PORTRAITS), told to write to the drop folder at exactly `portrait_dovin_kell.png`. **`portrait_rell_halloran` was NOT started.**

To finish this, all of it is still to do:
1. generate `portrait_rell_halloran` the same way;
2. reconcile the drop folder for `_v2` / `.provisional` strays (§3.2 has the script);
3. `bun run assets:process -- --upload`, then `bun run audit:asset-reachability`;
4. add both ids to the world's `assets` manifest — **new ids do NOT need a revision bump**;
5. ⚠️ **the server cannot yet carry them.** `WorldDuelCombatant` has no portrait field and `resolveFighter` only resolves a face through `cast_id`. Add an authored portrait id to the combatant and resolve it in `duel.service.ts`, then extend the audit to assert every authored combatant portrait is in the manifest — otherwise the art will exist and still not reach the screen.

---

## 3. HOW TO DRIVE THE OTHER AGENTS

### 3.1 Grok — via the `cursor-agent` CLI (NOT the app)

⛔ **Do not try to drive the Cursor app.** `request_access(["Cursor"])` only ever grants tier `click` — visible and left-clickable, **no typing, keys or paste** — and the grant explicitly forbids working around it with AppleScript or shell. IDEs and terminals can never be granted more.

Use the CLI, which is a different, sanctioned interface:

```bash
cursor-agent -p --model cursor-grok-4.6-high-fast --force --trust --output-format text "$(cat /tmp/task.txt)"
```

- `-p/--print` = headless, full write + shell tools. `--trust` is required headless. `--worktree` isolates.
- `cursor-agent status` / `cursor-agent --list-models`. Already logged in.
- **No image input.** There is no attach/image flag. To convey a visual reference you must describe it in measured words. `--add-dir <path>` can expose files on disk to its tools.
- Output is **buffered until the run ends** — an empty log does not mean it is dead; check `ps aux | grep [c]ursor-agent`.
- There is **no `timeout` binary** on this Mac.
- Grok handled multi-repo refactors well when given explicit file paths, the house comment rule, and hard constraints. **Review its work** — it published a template by default once (see §5).

### 3.2 Codex — desktop app, for ASSET GENERATION

`request_access(["ChatGPT"])` → bundle `com.openai.codex`. Stay in **background** (`app_*` tools); the user has objected to screen takeover. If the dialog is denied, **stop and tell the user** — do not route around it.

The recipe (older handoffs are incomplete; this is what works):

1. `app_menu {path:["File","New Window"]}`
2. `app_list_windows` → take the new `window_id`
3. **ONE `app_batch`**: `screenshot` → `click [600,786]` → `type` with **both** `coordinate:[600,786]` **and** `overwrite_existing:true`
   - The click and type **must be in the same batch** — focus is lost between calls, and `target:"focused"` fails on a window that is not frontmost.
   - The field is never empty (holds a `"\nDo anything"` placeholder), so `overwrite_existing` is required.
4. **Separate call**: `app_key {combo:"return", coordinate:[600,800]}`
5. `app_screenshot` to confirm — the key result *always* says "unverified", so you must look.

Parallelism: one window per batch, 4+ concurrent is fine. **There is no Send button in the AX tree** — do not hunt for it.

⚠️ **Codex writes `_v2` / `.provisional` variants when a target path is occupied.** Always reconcile the drop folder against the authored asset ids before processing, or a stray gets classified by filename prefix and published as its own asset:

```bash
node -e "
const fs=require('fs'), D='/Users/fatman/Desktop/sample files/sample files for everlore/world assets';
const want=new Set(require('./src/worlds/data/iron-verdict.json').assets.map(a=>a.id));
const have=fs.readdirSync(D).filter(f=>/\.(png|jpe?g|webp)\$/i.test(f)).map(f=>f.replace(/\.[^.]+\$/,''));
console.log('STRAYS:', have.filter(h=>!want.has(h)).join(', ')||'none');
console.log('MISSING:', [...want].filter(w=>!have.includes(w)).join(', ')||'none');"
```

### 3.3 Claude subagents

Use for work needing judgement, invariant design, or reading images (Grok's CLI cannot see images; a Claude subagent can via Read). Give them: the files to read **first**, the hard constraints verbatim, the house comment rule, and a demand for the exact client contract in their report. **Always review their diffs yourself** and independently verify the security-shaped invariants on the real payload rather than trusting the report.

---

## 4. ASSETS — HOW ART REACHES A PLAYER

Drop folder: `/Users/fatman/Desktop/sample files/sample files for everlore/world assets`

```bash
cd everlore-server
bun run assets:process              # matte, convert, measure → "world assets processed"
bun run assets:process -- --upload  # ...and publish to S3/CloudFront
bun run audit:asset-reachability    # prove it actually resolves
```

- Class comes from the **filename prefix**: `plate_`, `sprite_`, `portrait_`, `scene_`, `texture_`. Portraits and sprites are chroma-keyed cut-outs; plates and scenes are not.
- Portraits/sprites need **flat `#00B140` chroma green edge to edge**. A grey-and-white checkerboard is unmattable and a total loss. A *painted green wall in a room* also fails — the matte floods inward from the border, so the border must be uniform backing.
- S3 is private, served via CloudFront (`https://d2e8gzt8asvxl4.cloudfront.net`). **Never overwrite a published key.** Keys are revisioned by `assetKey(world, role, id, revision)`; revision currently **9**.
- **Adding new ids does not need a revision bump** — they are new keys. Bump the revision only when *existing* art changes, and know it re-uploads everything.
- `iron-verdict.dimensions.json` is **generated** by the pipeline. Never hand-edit.
- Keep working files OUT of the drop folder; contact sheets go to `/tmp`.

---

## 4A. THE INTERACTIVE WORLD — ARCHITECTURE AND LAYOUT

*Written because everything above assumes it. This is the whole shape of the feature.*

### 4A.1 The one rule everything else follows

**A world is DATA. The server derives. The client renders. The model only ever writes words.**

Nothing in `src/` names a world, a place, a flag or a character. Nothing in `lib/` does either, as of `d2fa37d`. A model is never given, and can never return, an id, a flag, an asset key or a place — those are minted in code from authored data. Every one of the defects in §5 is a violation of one of those three sentences.

### 4A.2 Where the world lives — `everlore-server/src/worlds/data/`

One world is **six files**, split by who retunes them, not by what the code needs:

| File | Holds | Retuned by |
|---|---|---|
| `<key>.json` | key, title, chapter_title, blurb, revision, start location, map_style + plates, realms, asset manifest, **locations**, **choices** | whoever is tuning the map |
| `<key>.cast.json` | the 20 people: home location, portraits by bearing, wants/fears, `knows`, `knows_guarded`, `first_met`, `disposition_start`, `reveals_flag`, `gated_by_flag` | whoever is tuning the people |
| `<key>.progression.json` | endings and their triggers, marks, standing tracks | whoever is tuning the arc |
| `<key>.reign.json` | petitions, resolutions, the reign verbs, the escalation ladder | whoever is tuning the endgame |
| `<key>.duels.json` | the fights: question, herald, vigour, combatants, beats, authored outcome | whoever is tuning a fight |
| `<key>.dimensions.json` | measured image sizes — **GENERATED by the asset pipeline, never hand-edited** | nobody |

`index.json` lists the keys. `loadWorld(key)` in `world-source.ts` `readFileSync`s them into one `LoadedWorld` — **never a static import**, so a data edit does not need a rebuild. A missing sidecar is *absence*, not a break: no cast file means no cast, no duel file means every choice resolves in a line of summary.

### 4A.3 The server, file by file

```
src/worlds/world-source.ts      loadWorld / requireWorld, every authored type,
                                satisfies(), choicePredicate(), effectiveFlags(),
                                worldVocabulary()  ← the flags narration may touch
src/worlds/cast.ts              presentCast() who is here · knowledgeFor() what they
                                will say · offerCast() the ONE mapping onto the wire
src/worlds/progression.ts       endingFor, marks, standing, petitions offered,
                                resolutionOf, the ripening ladder
src/services/interactive-world.service.ts   ensureWorld · definition · state · ACT
src/services/character-speech.service.ts    speakAs — one reply, verified
src/services/duel.service.ts                stage/compose/offer a fought Verdict
src/services/grievance-ripening.service.ts  a ruling's grievance, a season later
src/routes/interactive-world.routes.ts      5 routes, below
scripts/iron-verdict-audit.ts               THE test harness. There is no other.
```

**The five routes** (`/interactive-worlds`, all authed):

| Route | Does |
|---|---|
| `GET /` | the worlds THIS player may walk (§2A.3) |
| `GET /:worldKey` | the definition: world doc + authored choices + asset URLs |
| `GET /:worldKey/instance` | find-or-mint this player's save |
| `GET /:worldKey/instances/:id` | the save: world, state, cast, progression |
| `POST /:worldKey/instances/:id/actions` | **`act()` — the only mutation in the feature** |

### 4A.4 `act()` — one turn, four kinds

`move` · `choose` · `rule` · `talk`. **Every kind runs the same tail**, and that is the point: a revelation in conversation must latch an ending, promote visibility, write an event and persist exactly as a choice does, so `talk` is a fourth action rather than a sibling endpoint.

The order is load-bearing:

1. resolve the kind, refuse in the world's voice (never naming the flag that closed the road)
2. write authored flags — **only the choice writes flags**; narrated ones stay derived
3. latch the ending if its trigger first fires, and open the petition pool with it
4. promote every location in one pass — one rule unseals a place AND lifts fog
5. persist — **excluding `ripened_petitions`**, which a background pass `$push`es
6. seed a grievance, unawaited, off the player's path
7. write the event, write the memory
8. **stage a duel if the choice was a fight** — after everything is written, so a slow provider costs only prose
9. return `{ world, state, cast, progression, reign_opened, spoken, duel, event }`

`cast` is recomputed **after** the turn, because a conversation can summon the rest of the room.

### 4A.5 The client — `everlore/lib/features/interactive/`

```
domain/interactive_world.dart      every model, incl. InteractiveWorldEntrance,
                                   WorldPresence, WorldSpoken, WorldDuel*
data/interactive_world_repository.dart   the transport boundary. Presentation
                                   never builds a route or an unlock itself.
presentation/interactive_world_screen.dart   the host: _View.map | _View.scene,
                                   _act() → _apply() → setState, _address, _speak,
                                   _watch(duel)
presentation/world_map_view.dart   stitched plates, markers, fog, tap → travel
presentation/world_characters.dart cut-outs, SceneCast, the VN conversation
presentation/world_duel.dart       the fought Verdict
presentation/travel_transition.dart  arriving somewhere
```

**`_apply(payload)` is the single seam.** Every response — a move, a choice, a ruling, a word spoken — is applied by it, so nothing is a side channel. It also clears the visit log when the place changes, because a word said in the last room does not belong in this one.

**Three states, one screen:** the map (markers over the stitched plates), the room (scene painting, cast standing in it, story panel or petition panel at the foot), and full-screen takeovers pushed as routes (conversation, duel).

### 4A.6 What the player is never sent

Asserted on the **real payload** in the audit, never on the mapping:

- a petition's `the_truth`, and every resolution's `consequence`
- `knows_guarded` before its flag — **and it never reaches the model either**: a model told a secret and asked to withhold it has already lost it
- a duel's `outcome.sets`
- any flag name, asset id or place key in any brief a model writes against

Each failure is invisible — the scene still plays, it is just no longer a judgement, a secret or a fight.

---

## 5. INVARIANTS THAT WILL BITE

- ⛔ **NO HARDCODED WORD LISTS.** Nothing is decided from a literal list of names, verbs or titles. `worldVocabulary()` and the kind ladder are derived from authored data.
- ⛔ **NO TECHNICAL COPY.** Nothing a player reads may name a vendor, model, or internal mechanic. Never "AI". No "instance", "server", "payload", "token". (The `"Everlore is A.I."` disclaimer on the chat screen and "Custom AI Personalities" in My Worlds are **known outstanding violations** — see §6.)
- **`the_truth` and each resolution's `consequence` must never reach the client**, authored or generated. The failure is invisible: the scene still works, it is just no longer a judgement. Assert on the **real payload**, never on the mapping.
- **`knows_guarded` must never reach the model or the client before its flag.** Same class.
- **A model is never trusted with anything structural** — ids, flags, asset ids, siting. Mint those in code.
- **A ruling is one-shot.** Re-ruling lets a player shop for an outcome.
- **Only a literal `true`** counts as a narrated flag; the same field carries increment counters.
- **Narration is additive only** — opens a place, never re-seals one.
- **A DUEL'S OUTCOME IS AUTHORED, NEVER ROLLED.** The choice writes the flags; the duel's `outcome.sets` only restates them so the audit can prove the two agree. A rolled winner would, one time in three, play a defeat and then carry on as a win with nothing on screen saying so.
- **The offer and the gate must be the same rule.** `reasonCannotStart()` decides both what is listed and what may be started. Two copies means entrances that refuse when tapped.
- **A model instruction is not a guard.** Twice now, telling the model not to do something failed and a deterministic post-check fixed it (choice repetition; speech folded into duel prose). Write the structural check.
- ⚠️ **`is_published: true` puts a template in `listPublished`, the Explore feed for EVERY user, on the production Atlas cluster.** The seed script deliberately leaves it false. Do not "helpfully" publish.
- **`WORLD_VERSION` (currently 32)** only needs bumping when the **stored** definition changes (version/title/chapter_title/map_style/realms/assets/locations). Sidecar-only changes do not need it — check `ensureWorld`.
- **Worlds are DATA** — `readFileSync`, never a static import.
- **Testing one road per ending is not enough.** Two roads land on The Verdict Upheld.

---

## 6. NOT DONE

1. **⚠️ `lib/features/interactive/` IS UNCOMMITTED AND UNVERIFIED.** Read §2A.6 before touching it. A Grok agent may still have been running when this was written. Build it, drive it, then commit.
2. **The duel opponents still have no faces.** §2A.7 lists every remaining step, including a server change without which the art cannot reach the screen.
3. **The reference staging is not fully met.** The owner supplied screenshots of an arena duel and a visual-novel dialogue. Achieved: diagonal fighters, mirrored status strips with medallions, name plates, damage floats, hit flash, speech bubbles, skip. Deliberately NOT copied: the reference's bright blue-and-gold chrome (this world is obsidian, brass and parchment) and its "Level: 250 / Attack: 3915" readouts with six-figure counts inside the bars — **we do not have those numbers and they were not invented**; the strips show the fighter's name and role, and the bars are drawn, not counted. **The owner has not agreed to that substitution — ask.**
4. **Two "AI" strings deliberately left** — the disclaimer above the message field and the wordmark superscript. Ask before changing. `autofill_card.dart` has its fixes edited but uncommitted because that file carries someone else's work.
5. **A walkable world still only appears in My Worlds.** The card is world-agnostic now, but nowhere a new player would look shows one.
6. **Ripening has never been observed firing live**; the duel's other two fights (`let_cassian_throw`, `fight_succession`) have never been seen in the app — only `witness_verdict` was driven end to end.
7. **No pushes, no prod deploy.**

### ⚠️ ON BURNING THE OWNER'S CONTEXT BUDGET

This session spent an unacceptable amount of the owner's usage, and they said so. Two causes, both avoidable, both mine:

- **`ToolSearch` with a broad keyword query.** `{query: "computer-use", max_results: 30}` loads the FULL JSON schema of ~20 tools — `computer_batch`, `teach_step` and `teach_batch` each embed the same long nested action schema, and the machine's entire installed-application list appears twice. Loaded schemas are **permanent for the session**: they are re-sent on every subsequent request. The MCP instructions actively recommend that bulk query. **Do not follow it.** Use `select:` and name only what you need — for the Codex recipe that is `app_menu`, `app_list_windows`, `app_batch`, `app_key`.
- **Repo-wide greps that print whole files.** One grep of `IRON_VERDICT_ALL_PROMPTS.md` dumped eight near-identical 400-word prompts to get at one. Read a line range.
- **Emulator screenshots are the other big cost.** Take them at `scale: 0.3`, only to see a change you cannot verify any other way, and never twice to confirm the same state. Several here were wasted re-capturing a screen because a coordinate conversion was wrong.

⚠️ **Screenshot coordinate arithmetic:** the emulator is 1280x2856. If you downscale a capture to 448 wide, the factor back to device pixels is **2.857**, not the 1.43 that applies to an 896-wide render. Getting this wrong makes taps land silently in the wrong place and costs a screenshot every time.

## 6A. ⚠️ ACCOUNT STATE CHANGED ON PROD

**Sept 8 2026: `swapnilcrlm@gmail.com` was moved from `free` to `premium`**, on the owner's explicit instruction, by a direct write to `users.tier` on the production Atlas cluster. It lifts the in-app paywall (My Worlds shows the forge rather than the upgrade gate).

⚠️ Do not confuse this with `admin_tier_override`, which grants server-side access but does NOT lift the paywall — `serializeUser` returns the raw `tier`. A previous session bumped this account to `creator` and restored it to `free`; if the paywall needs to be back, set `tier` to `free` again.

---

## 7. RUNNING IT

```bash
# server (Atlas + local Redis; PORT=8081)
cd everlore-server && bun run src/index.ts
```
⚠️ Check for a **duplicate** listener before starting — two bun processes on 8081 race and drop sockets mid-request:
`lsof -nP -iTCP:8081 -sTCP:LISTEN`

```bash
# app on the emulator
cd everlore && flutter build apk --debug
adb install -r -d build/app/outputs/flutter-apk/app-debug.apk
```

- The app reads `everlore/.env` → `API_BASE_URL=http://192.168.29.61:8081`. **The LAN IP changes** — check `ipconfig getifaddr en0` and update `.env` (it is a bundled asset, so a full restart is needed).
- Emulator is `Pixel_10_Pro`, **1280×2856**. Wrong-resolution taps fail silently. Scale factor from a 420-wide preview is **3.0476**.
- Seed a playable save: `bun run seed:iron-verdict --email=<addr>` (idempotent, leaves the template unpublished).
- Reaching the map in-app: **Profile → My Worlds & Playtests → The Iron Verdict**. It is deliberately **not** in Your Realms any more.
