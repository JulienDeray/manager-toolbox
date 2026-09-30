# Semester review: after the review (Phase 5)

The full procedure behind Phase 5 of [`semester-review`](../SKILL.md). Run on request, after the
review meeting.

1. **Read the live notes** from the meeting page first; the manager takes them inline under the prep
   bullets during the meeting. They are the source for "as delivered", reactions, and agreed goals.
2. Update `people/{slug}.md`:
   - New `## {Cycle} Review ({Month Year})` section at the top: strengths and improvement areas **as
     delivered**, each with an **Evidence:** line, each growth area with its **Framing given:**
     one-liner, then a `### Reactions & Agreed Goals` block (how the report reacted, 🎯 goals with
     the support you committed to).
   - Move the prior review section under `## Previous Review`.
   - Update the patterns/themes table.
   - Prune the `## Observations Log`: delete entries incorporated into the delivered review (their
     substance now lives in the review section); keep still-`[open]` entries.
3. If you use `intake-for-me`, spawn `intake-for-me sweep {page URL}` as a background sub-agent to
   pick up inline capture markers dropped during the meeting.
4. If the notes don't answer them, ask the manager two gap questions before drafting the recap (the
   answers shape its tone): how did the person react to the growth feedback, and did they give any
   upward feedback?
5. **Draft the recap message** in your chat tool as a draft DM to the person; **never send it**, the
   manager reviews and sends by hand. If drafts aren't supported, output the text in chat. Recap
   rules:
   - Same shape as a 1:1 summary: `🚀 Review Summary: {YYYY-MM-DD} 🚀`, flat bullets with indented
     sub-details, short.
   - **Room-only content**: nothing from peer feedback, cross-mentions, chat sweeps, or private prep
     sections; only what was said in the meeting.
   - **Their material first**: lead with what they were proud of and their own ideas, so it reads
     "we built this together", not "here's your grade".
   - Include the growth-area **framing one-liner verbatim**; it's the carryable phrase.
   - Close with the sketched goals, the support the manager committed to, and the development-plan
     timeline.
6. Mark the meeting Done in the notes tool.
7. Remind the manager that the next cycle's development plan should follow within a few weeks;
   `create-pdp {Person}` builds it from this review's outputs.

