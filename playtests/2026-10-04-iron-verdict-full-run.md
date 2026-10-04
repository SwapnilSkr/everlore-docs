# Iron Verdict: full playthrough audit, Oct 4 2026

One fresh walk, played as **Nara Voss** on the emulator against a local server, from the overture to an ending (**The Bloodless Crown**) and one reign ruling.

- **Route:** overture → Watch the Verdict → pick up Cassian's mark → read the writ → swear a challenge → tell the Arbiter → leave to read the rolls → give the writ to the Court → ending → Reform → rule "The Seal That Reads True".
- **Size:** 15 steps and 11 character lines.
- **Spend:** Luna-low speech, about ₹1.
- **Cleanup:** the test walk and its 6 achievements were deleted afterwards. The owner's save was not touched.

Severity: **P0** breaks the story or misleads, **P1** clear defect, **P2** clutter and polish.

## Guidance and the map

- **P0 — The start place is hidden.** Nara starts at the Sundial Market, but the map opens centred on the Ring with *Travel* selected. The market is off-screen and authored `sealed` until `verdict_witnessed`, so her own start place looks locked. Her opening deed (*Listen to the Ring from the stalls*, the tracked goal, marked "Here") can only be reached by finding the market by hand. The map must open on the lead's place, and a lead's start place must never be sealed to them.
- **P0 — Finding a destination means panning by hand.** The tracker names a place ("The Sable Court · 1 road away"), but the map never pans to it and tapping the tracker opens the journal. Finding the Sable Court took 6 manual pans. The goal marker sat under the bottom place card (the market) or under the tracker's scrim (the Sable Court). Fix: tapping the place line centres the map on that marker, and markers are kept clear of the top and bottom chrome.
- **P0 — The reign has no guidance.** After the ending the app hides the tracker. The server still tracks pre-ending goals, with "Hear the petitions" ranked last. Petitions are found only by wandering into them. After an ending, petitions should be the tracked goal and the tracker should stay on screen.
- **P1 — Grouped goals point the wrong way.**
  - "Watch the Verdict in the Ring" is shown to Nara, whose own version of that goal happens at the market.
  - "Find out who wrote the verdict" points to Hollowforge (2 roads) right after winning leave to read the abbey rolls, because the goal's target is the nearest place rather than the one the deed opened.
- **P2** Map marker labels are cut off ("The Sundial Mar…").

## Moments and achievements

- **P0 — Goal and achievement moments play over the ending.** "Goal complete", the achievement card and then "NEW GOAL · Find out who wrote the verdict" all played over the ending screen. A new goal after the story has ended makes no sense.
- **P0 — Two achievement systems with conflicting text.**
  - The ending lists five *marks* from the older progression file (Sand and Silence, A Champion's Word…). The new achievements have the same names but different text.
  - "A Champion's Word" says Cassian "told you a thing he had told no one". As Nara, I never met him.
  - Pick one system, and write the text per lead or neutrally.
- **P1 — Some moments don't appear.**
  - At the market, reading the writ completed a goal and unlocked an achievement (recorded on the server). Neither the moment nor the card appeared.
  - At the ending, "Getting somewhere" never showed as its own card.
  - At the Ring the goal moment and the card both played, but overlapped.
- **P1 — The goal moment dims the reply.** It dims the whole screen for about 5 s, including the character's reply as it streams in. One fully undimmed frame flickers between "Goal complete" and "New goal".
- **P1 — "Seven names on the road" fires too early.** It unlocked after 4 places were visited, because it counts places *revealed*, not *visited*. Fix the condition or the text.
- **P2** A ruling gets no outcome moment: the panel closes back onto old room text.

## Conversations

- **P1 — Every deed opens the Herald's conversation**, even ones that have nothing to do with her (picking up Cassian's paper). You must tap *Step away* after every action.
- **P1 — Repeated outcomes.** After the Verdict the same fact appears three times at once: the echo line, the scene text, and the Herald's "The loser was carried down the stair." The echo check compares the reply against the deed *label*, not the outcome text the player has just read.
- **P1 — A wordless greeting is dropped silently.** "Speak with Arbiter Vell again" started a speech that was never shown, never stored, and logged no flagged, failed or end line. The typed question that followed worked fine.
- **P1 — Figures float after the ending.** In conversations after the ending, both figures are about 260 px too high and their feet hang in mid-air.

## Room, scenes and text

- **P0 — Text written for the wrong lead.**
  - The Ring scene after the challenge says "The woman under the bronze dial can read a seal without breaking it." Nara *is* that woman.
  - The first reign petition, "The Seal That Reads True", makes Nara a *party*, so she is asked to judge herself in the third person.
- **P1 — Scenes don't change.**
  - After the Verdict, and even after the ending, the market still shows Nara's pre-fight scene and still offers "Listen to the Ring from the stalls". `hear_the_ring` needs `forbids: verdict_witnessed`.
  - The place card's "A new clue waits among the lantern stalls" never changes.
- **P1 — Echo lines replace what happened.** After a conversation, the room's echo line says "Spoke with Arbiter Vell." in place of what was actually done there. The scene text doesn't change after the Arbiter summons the Court.
- **P2 — Buttons are cut off.** The bottom button of the room panel (Work the place, Speak with … again) is clipped with no hint to scroll.
- **P2 — The turning point looks like any other choice.** "Burn the writ" looks the same as routine choices. Only the journal carries its warning.

## Fights

- **P2** The "80 / 100" count is drawn over the bar's segment ticks in dark text on brass, so it's hard to read.
- **P2** At the end of a fight the winner's bar disappears, leaving the loser's "0 / 100" strip alone at the top.
- **P2** The result card stacks three blocks, and the italic preamble is still literary prose. One plain outcome line would do.

## Ending

- **P1** The ending screen is a wall of about 600 words: tagline, summary, a long paragraph, six bullets, five mark cards, then three buttons. "Reform" (which starts the reign) isn't explained.

## Walk list

- **P2** The "You've wandered here before" sheet names walks "Walk 3 / 2 / 1" with no lead or place, while the Walks tab says "1 walk".

## Server only

- **P1** The silent speech drop above: a speech job that ends without any terminal log line.
- **P2** Deleting a walk tries to delete Pinecone namespaces that walks never create. Each call returns 404 and is logged as a failure, 8 times per walk.
- **OK** No `[warn]` or `[error]` lines during play. No speech was flagged. Every act returned 200. Achievements were recorded for every unlock.
