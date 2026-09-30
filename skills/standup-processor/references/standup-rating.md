# Stand-up rating + improvement suggestions: detailed procedure

> Capability 4 of the `standup-processor` skill. Written in terse `full` style, like the
> internal Slack wrap-up (ephemeral, read once by the operator at the end of the run).

## The rubric

Six dimensions. Each is scored **pass / partial / fail** against the transcript, with a short
quote or example as evidence; never a bare score with no grounding. The rating scores the
meeting's format, never an individual; quotes carry no names.

**What the rubric scores.** Many teams deliberately keep the stand-up discussion-oriented: a
strict status round-robin adds no information anyone couldn't read off the board, while
unbounded detail-dives are a real cost. So the rubric scores the *intended* format: a
high-level overview plus solution-oriented collaboration, with detail drift **contained or
moved to a breakout**, never rewarded for being absent. Don't score recitation as discipline,
and don't score discussion as drift. If your team has settled this tension differently (e.g. in
the `team_agreement_page` read in Step 0), adjust dimensions 1-4 to match; they are your team's own rules, scored back
at the team.

**From the team's stand-up format rules** (a time cap like "15 minutes max" is deliberately not
scored: a transcript alone can't reliably establish elapsed time):

1. **Epic-level led, not ticket-by-ticket.** Did the stand-up open with and organise around
   epics (what's closest to done, this week's priorities), rather than working
   ticket-by-ticket?
2. **Work-centric, not round-the-table.** Did the conversation walk the board and follow the
   work, rather than going person-by-person regardless of whether they had something
   board-relevant to say? A disciplined round-robin status recitation is **not** a pass:
   recitation is the format this rubric rejects.
3. **People flagged needs, not status.** Did updates surface who needs help, what's blocked,
   what's worth discussing, rather than reciting "did X, will do Y" with no ask attached?
4. **Breakouts deferred, not solved live.** Collaborative discussion is the point of the
   format; the fail mode is an **unbounded whole-team deep-dive**, not the existence of
   discussion. A dive that stays brief and contained, or gets explicitly parked for an
   after-stand-up breakout with the people involved, passes; one that derails the whole-team
   session doesn't.

**From stand-up-discipline research** (Martin Fowler's stand-up pattern language, Jason Yip's
obstacle-tracking pattern):

5. **Reporting to the team, not to the manager.** Were updates framed as talking to teammates
   ("I need X from you") rather than reporting up ("here's my status", implicitly addressed to
   the manager)? This is the most-cited failure mode in the research: a stand-up that's technically
   peer-attended but functionally a status report to one person isn't serving its purpose.
6. **Blockers tracked to resolution, not just voiced.** Did a raised blocker get an owner or
   next step (escalation, a linked ticket, "I'll ping their lead") rather than being mentioned and
   left hanging? Cross-reference Capability 1's output: if a blocker triggered a comment, new
   ticket, or escalation there, that's evidence of a pass here.

## Scoring method

- Read the full transcript once with the rubric in mind before scoring; don't score
  line-by-line on the first pass or you'll miss cross-cutting patterns (dimension 5 is about
  the *whole* transcript's framing, not any single line).
- Score each dimension independently; a fail on one doesn't imply a fail on another.
- Quote the transcript directly as evidence. A fabricated or paraphrased-beyond-recognition
  quote is worse than no quote.
- If the transcript is too short or thin to judge a dimension (e.g. no blockers were raised at
  all, so dimension 6 has nothing to score), say so explicitly rather than forcing a score:
  "N/A: no blockers raised today", not a guessed pass.

## Improvement suggestions

Generate **up to 3** suggestions, tied to whichever dimensions scored `partial` or `fail`. This
is a **ceiling, not a target**:

- If fewer than 3 dimensions scored below `pass`, suggest fewer.
- If a dimension scored `fail` but no genuinely actionable fix comes to mind beyond "do the
  opposite", don't manufacture a suggestion to fill the slot; silence is fine.
- Each suggestion should be specific to *today's* transcript, not a generic reminder of the
  rule. "3 of 4 updates opened with a ticket ID before mentioning the epic; try leading with
  the epic name instead" is useful; "remember to be epic-level" is not.

## Delivery: chat output, end of run

Print the rating and suggestions **in the conversation, as the final section of the run's
closing message**, after every other capability has completed (Jira writes executed and
verify-read, Slack drafts handed off). It is the last thing the run outputs, so the operator
reads it once everything is done, never mid-run where it would get scrolled past by report
tables and drafts.

**No Slack at all**: not a post, not a draft, not a DM. A self-DM triggers no notification, so
it goes unread; do not add one.

Not posted to any channel, not written to Confluence, not tracked run-over-run: each rating is
self-contained. If a trend view is wanted later, that's a deliberate future change (a wiki log
appended weekly), not something to add silently here.

## Format

```
**Stand-up rating: DD-MM-YYYY**
1. Epic-level led: [pass/partial/fail], "[quote]"
2. Work-centric: [pass/partial/fail], "[quote]"
3. Needs flagged: [pass/partial/fail], "[quote]"
4. Breakouts deferred: [pass/partial/fail], "[quote]"
5. To the team, not up: [pass/partial/fail], "[quote]"
6. Blockers tracked: [pass/partial/fail], "[quote]"

Suggestions:
- [suggestion, if any]
```
