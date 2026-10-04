# Iron Verdict: nine playthroughs, Oct 4 2026

Played on the emulator after W11 and W11 fix-1: three walks for each lead.

| Run | Lead | Route | Ending |
|---|---|---|---|
| C1 | Cassian | Fight → Chainhouse (key, train ×2) → Stair → Market → Fane → Court → Hollowforge (seal, forge) → known roads back → Court | Bloodless Crown, rewound and retaken, then 2 rulings |
| C2 | Cassian | Fight → writ read → swear → burn at the Fane → Throw the Verdict | Death sheet → rebound as Nara mid-reign |
| C3 | Cassian | Fight → Market → Hollowforge → Marblegate → Ravensgate → Thornhollow | Broken Ring, then 2 rulings (one sent to the Ring) |
| N1 | Nara | Market → Ring (mark) → Hollowforge → Marblegate → Ferryman's Post (dead end) → train → swear → burn → fight at Str 5 (lost) → rewind → train to 9 → fight | Verdict Upheld (lost fight), then Iron Verdict |
| N2 | Nara | Market → Ring → south → Thornhollow pact → Rimegate → Grimsdal Mine (dead end) → Thornhollow | Broken Ring, then 1 ruling |
| N3 | Nara | Conversation stress test (an attempt to make a character break role, an action, history) → Fane → Court, mid-walk rewind | Bloodless Crown; her own petition is correctly hidden |
| S1 | Sereth | Court route; family questions to the Duke | Bloodless Crown; ruled against Nara, then spoke to her |
| S2 | Sereth | Ring → burn → gated fight (Str 2) → let Cassian throw | Verdict Upheld |
| S3 | Sereth | Court leave → Ring trust → south → Thornhollow | Broken Ring |

**Server health:**
- 195 acts, all HTTP 200.
- 72 speeches ended; 1 was flagged and 0 failed.
- No warnings or errors in the main or worker logs.

**Cleanup:**
- All 9 walks were deleted by id, and their achievements were cleared.
- The owner's save was restored.

**What works:**
- The map opens on the lead's place.
- Tapping the tracker opens the map on that marker.
- The ending moments queue properly.
- The ruling moment plays.
- Rewinding and retaking a choice works.
- Quick travel along known roads works.
- The walk list shows the lead and place.
- The reign tracker reaches every petition at a place with a scene.
- Characters stay in role when the player tries to make them break it.
- The lead is never asked to judge their own petition.

---

## P0: blocks a route or an ending

1. **Choices at a place with no scene can't be taken.** The app shows choices only inside a room, and a place with no scene has no Enter button. Three choices are affected:
   - `hire_the_ferry` at Ferryman's Post;
   - `mine_oath` at Grimsdal Mine;
   - `buy_introduction` at Spice Vaults.

   This cuts off the whole south chain (Blackreach → Landfall → Gilt Exchange → Casa Verrine → **selling the writ**) and the north chain past the Rimegate. The tracker sends players there anyway ("Work a shift in the Grimsdal mine · Here", with nothing to tap). `audit:iron-verdict` calls these choices reachable because it doesn't model the client.
2. **Petitions at places with no scene can never be ruled:** Harrowfield, Weeping Quarry, Coldwatch Tower and Ferryman's Post. The reign tracker will route players there.
3. **Cassian can never see Verdict Upheld.** "Throw the Verdict" meets that ending's trigger, but the death sheet shows instead ("The sand has you… take it again, stronger"). That treats a deliberate throw as a lost fight. Rebinding then drops the new lead straight into the reign, with no ending screen, cost or introduction.
4. **Cassian can't fight the succession honestly.** `fight_succession` is `not_for_leads: cassian_vale`, so the champion has only "Throw" and can never reach the Iron Verdict ending.
5. **The goal points at a burned writ.** After "Burn the writ", the new goal is "Tell Arbiter Vell the writ is dated early", and that choice stays offered. After the ending, a rebound Nara is still offered "Read the paper in your coat". `name_the_date` and `keep_the_days_writ` should forbid `writ_burned` and `succession_passed`.

## P1: wrong or confusing in normal play

**Room panel**
- The last row sits under the bottom fade, and the scroll springs back as soon as the finger lifts. Rows such as "Train…" and "Speak with …" can only be tapped mid-scroll. Once, a reflow moved the tap from "Fight the succession" onto "Let Cassian throw the Verdict", which is a turning point.
- When a choice is pending, every button shows a spinner, not just the one tapped.

**Echo line**
- After a drill, a burn or a talk, the previous place's echo stays at the new place ("You haggled in the stalls…" shown at the Ring). This was seen three times.

**Map**
- "The road is known" appears on any open place that isn't next door and hasn't been walked. It has no Travel button and no route hint.
- Places two or more roads away mean hunting the next hop across a large, foggy map. The tracker names the final place (sometimes one not yet revealed), not the next road.
- The conversation panel's own tracker has no `onLocationTap`, and it stays hit-testable after "Step away". Tapping the place line then opens the journal instead of the map (`world_characters.dart:601`).
- "World map" in the room often needs two taps.
- **W11 regression:** the new boundary margin lets the camera pan past the edge of the map painting, showing a black strip.
- Markers under the header scrim, or under the place card, can't be tapped or read.

**Speech and character knowledge**
- Characters don't know who the lead is to them:
  - Gaunt and Hob treat Cassian, the Ring's champion, as a stranger.
  - The Duke tells his niece Sereth "We've met before".
- What happened isn't passed into later conversations:
  - Iselle asks to see the seal that Sereth showed her one step earlier.
  - Nara greets Sereth warmly right after Sereth ruled her a forger.
  - The Herald mentions the folded paper on the rail before the Verdict that leaves it there.
- The echo check misses a second-person retelling:
  - "You chose to fight the Verdict."
  - "This verdict was dated before the fight."
  - A flagged line ("You let Cassian throw the Verdict…") is still sent to the player as the reply.
- An action typed in *asterisks* is shown as "You asked: '…'". The Earlier words sheet leaves the action out and is hard to read.

**Strength gate**
- "Fight the succession" unlocks at Strength 5, but Halloran is rated 9. A Strength 5 lead loses for certain.
- At Strength 2 the button looks available, and the gate shows only as a notice after tapping.
- The required strength is never shown.

**Fights**
- "Raise your shield" changes nothing, including in a fight the player chose to lose.
- The fight narration says "Dovin" while every other screen says "Kell".
- On the result card the labels are swapped: the decision has no label, and "What was decided" sits above Kell's fate.

**Turning point warnings**
- Every turning point, including "Throw" and "Let Cassian throw", says "If it goes against you… train your arm".

**Text for the wrong lead or the wrong moment**
- Cassian sees "Cassian is not a prisoner, but he cannot leave", "Yours is in your coat" and "A folded paper with your mark…". His Ring scene still says "You are still on the sand" after he has left and come back.
- Sereth, who already holds the writ, is steered to "the woman under the bronze dial" and "Pick up the mark".
- Sereth's Verdict Upheld copy says "The Court owns your calendar", yet she is the Court's heir.
- Verdict Upheld says "you are paid for it" on the burn-and-throw route, where nobody pays the lead.
- Broken Ring promises a barricaded Court and a shut Marblegate. Neither happens in the reign.

**Deeds credited wrongly**
- "Leave the city under the wall" tells a Cassian who never held the writ "You carried the sealed ruling…" and awards "Nothing left unread". The lead also doesn't move.
- The journal's Done list ticks "Ask Gaunt what the unused key opens" for a Cassian who remembered the stair himself.

**Getting stuck**
- Sereth gets the Court's leave to read the abbey rolls, but the Abbey stays hidden until someone asks the Arbiter about the rolls. Nothing points her there.

**Rebinding to a new lead**
- The new lead's prologue is skipped.
- The dead lead's progress flags carry over.
- The tap on "Walk as …" shows its spinner on "Take the fight again".

**Rewinding**
- Rewinding lands on the map instead of the room.
- A held ending card plays after rewinding past that ending.
- The step count on the clock button doesn't drop.
- Tapping one step puts a spinner on every step.

**Endings**
- "Achievements earned" lists badges from an ending that was rewound away, and leaves out ones the account already had from earlier walks.
- Expanding "What changed" pushes the buttons off-screen.

## P2: polish

**Text and labels**
- "Chapter I · The Bloodless Crown" names one of the four endings, on the Walks card and in the map header.
- The overture's "The Hinge" doesn't say how to rewind.
- The lead picker's "Turn them. Stand with one." is unclear, and the stats are never explained.
- "Ask Gaunt what the unused key opens" (the goal) doesn't match "his spare key" (the button). The topic says "door on your belt", which should be "key".
- "Trace the matrix" is jargon.
- The Arbiter's line "I named it myself" leaks the internal wording of the deed.
- The topic "What became of the old rolls?" stays offered after it has been answered.
- The rewind sheet labels Nara's thrown fight "FATAL" in red.
- The road card "The road to The Sable Court" is tagged with the place the lead left.
- "Hob Cardew works for the Ring, not for strangers" doesn't say that winning Cassian's trust unlocks it.

**Feedback**
- Training shows no "+1 Strength".
- The standing chip ("−2 · 1 more") is unexplained, and the Standing strip shows raw numbers.
- "Fight the Verdict" and "Take the known road" give about 4–10 s of no feedback before anything happens.
- The map flashes a grey plate for about 1 s before the painting loads.

**Layout**
- Walk rows wrap mid-phrase ("1d / ago").
- The map's opening zoom hides the places next door.
- The petition panel covers the petitioner. With three parties, the third ruling is clipped.
- A two-line ruling label overlaps the panel, hiding the outcome.

**Places and art**
- The death sheet shows a stray small figure of the dead lead, and repeats its own sentence.
- The Ravensgate room is empty, and the lead's figure is missing there.
- Hob's figure is missing when Nara arrives.
- A hard seam runs across the map painting in the north.
- Sereth's opening hall names Aldric and Barran, but neither is present.

**Accessibility and achievements**
- Controls behind opaque sheets are still in the accessibility tree.
- Some visible markers are missing from it.
- The achievements progress bar reads as a bare "0".
- Locked achievements give no hint and are barely legible.

**Conversation input**
- Enter adds a newline instead of sending.
- The first tap on Speak while the keyboard is open is swallowed.

## Server only

- The `http.request` log shows `durationMs` of about 0.01 for every act, so act latency can't be read from the logs.
- Character replies log no usage or cost line (only fight staging does), so the spend on a walk can't be measured.
- Speech that is flagged is still delivered to the player as the reply.
