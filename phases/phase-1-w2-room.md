# W2 — Arriving and the room: design brief

*Phase 1, workstreams 2 + 2a. Written Oct 1 2026. Built from the read-only audit `audits/a2-room-audit.md` (Grok), which is the measured spec: apply its §3 defects 1–11 with the numbers it gives, keep everything on its "Leave alone" list, and build its §4 target layout. This file records the orchestrator's decisions on the audit's open questions, plus the veil (§W2a in phase-1.md).*

## Decisions

1. **One sequence, three stages, no overlap:**
   - **Veil** (W2a): mist plus the place's name and line, covering only the wait for decode, bounded at 8s as today.
   - **Walk**: starts only once the destination is decoded. 2600ms, plays above the mist while the mist crossfades out (600ms `easeOutCubic`), holds 400ms, then crossfades 500ms into the live room.
   - **Room**: the parchment fades in (600ms `easeOutCubic`), and the people's 560ms entrance starts here, not under the veil.

   The veil's words exit with the mist. A known-road haste is the veil and then the room, with no walk.
2. **Who opens a conversation on arrival:**
   - A plain `move` settles on the room and never opens the chat.
   - When the server returns `spoken.initiated`, or a fight just ended (W4: never left hanging), the speaker's **one new line** appears as the current beat. It shows in the conversation paper with the same collapsed-history layout, never the full transcript.
   - The client stops picking `cast.first` as a speaker on its own (`_personToAddress` fallback). The server decides who speaks.
3. **Stat chips** (STR/CHA/LEAD/LV and standing) leave the parchment and become one muted line under the place title in the top chrome, at 11px with 1.0 tracking. They aren't dropped.
4. **`reign_opened`** (the ending's opening beat, today ignored by `_apply`) is shown as the current beat with one event inscription above it, the first time it arrives.
5. **Events** are one-line inscriptions (13px italic, muted): the newest stays, and more than three collapse into "Earlier words".
6. **Typewriter:** show the finished line with a 280ms fade, and drop the 18ms append. The model's long paragraph becomes a calm paragraph.
