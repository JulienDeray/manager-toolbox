# Gather context: detailed procedure

> The read-only half of the `planning-agenda` skill. Collects the context sources, buckets them
> deterministically, and hands the result to [agenda-message.md](agenda-message.md). Everything
> here is **read-only**: the skill's only write is the Slack post. The bucketing described in G8
> is mechanical and identical every week (helper script recommended; see the README caveats);
> otherwise apply the rules by hand, carefully.

## Contents

- G0: Planning week + slot
- G1: Last week's focus review
- G2: Backlog-review outcomes
- G3: People & availability
- G4: WIP/flow trend *(optional)*
- G5: Migrations needing attention
- G6: Step-back schedule (run this right after G0; a step-back week short-circuits G1/G2/G4/G5)
- G7: Direction banner (skipped on step-back weeks)
- G8: Compute
- Notes & edge cases

Config is already loaded (Step 0), and the whole gather happens before anything is posted.
**Determine the planning week first**: it parameterises several sources (the availability window
and the migration nudge-due cutoff).

## G0: Planning week + slot

- **Slot** = the argument if given, else the `planning_slot` from TEAM_CONTEXT.md.
- **Week** = Monday to Sunday of the week containing the slot. Resolve the slot in
  `team_timezone` so a scheduled wall-clock run sits correctly before the slot across DST
  switches.

## G1: Last week's focus review

1. **Live holders (the normal path).** The agenda runs before `planning-processor` does its
   weekly clear-and-reapply of the focus label, so a live query still returns **last week's**
   focus set:

   ```
   project in (PROJ, OPS) AND labels = "prio::now" ORDER BY priority DESC, key
   ```

   **Run it via the Jira REST API, not the MCP search tool**: some MCP searches silently cap
   bulk results at a handful of rows, and a focus set of 6-8 already crosses that. The failure
   is quiet in a nasty way: a short result still looks like a plausible focus list, so the
   agenda would under-report last week's focus rather than error, and the fallback below would
   not trigger either (the query returned *something*). Cross-check the count against the
   previous week's wrap-up post before composing the agenda.

   For each holder, fetch the issue **with its changelog** and capture: key, summary, current
   status category, assignee, the date of the most recent status transition, and **`tagged_on`**,
   the date the item received the focus label last week. The reliable source for `tagged_on` is
   the most recent prior planning comment (`planning-processor` posts one, with the
   `planning_comment_prefix`, on every focus item); its date is the prior planning date. If
   absent, use last week's planning date.

2. **Fallback: the snapshot was already cleared.** If the live query returns **this week's** set
   (someone ran `planning-processor` first) or is empty, do **not** treat it as last week's
   focus. Reconstruct last week's set from the most recent prior planning comments across the
   projects: the items carrying the latest planning-date comment *were* last week's focus.
   Review those for status now vs that date. **Never re-apply labels or modify
   planning-processor** to work around this.

3. **Deep check before calling anything stalled.** An epic's own status rarely moves; the work
   moves in its children and its description. (An epic can sit "In Progress" for months while
   three children go Done and its checklist is ticked; a status-only check calls it stalled.)
   For every holder that is not Done and has no own transition after
   `tagged_on` (the would-be-stalled set, typically 0-3 items), gather movement evidence:

   a. **Child movement** (one JQL per item):
      `parent = <KEY> AND (status changed AFTER "<tagged_on>" OR created > "<tagged_on>")`.
      Each hit becomes an evidence entry: key, what happened ("moved to Done", "created"), date.
   b. **Own description edit** (no extra call): scan the already-fetched changelog for
      description changes dated after `tagged_on`. If found, record the date and a one-line
      reading of the diff (e.g. "checklist: 3 items ticked"); the agenda quotes it as the *how*.
   c. **Child description edit** (fallback, the checklist-in-a-subtask case), only when a and b
      are empty: `parent = <KEY> AND updated > "<tagged_on>"`, fetch the changelogs of those few
      children, keep only description changes. Ignore rank/label churn.

   **Always record the deep-check result explicitly**: an empty child-movement list means
   "checked, nothing moved" and keeps the stalled verdict trustworthy; an *absent* check means
   "not checked" and must be flagged as such in the agenda ("status only, deeper check
   missing"). If a deep-check query errors, flag rather than guess.

Bucketing: **closed** (Done) / **advanced** (own transition, child movement, or description edit
after `tagged_on`, with a human-readable evidence trail) / **stalled** (deep-checked, nothing
moved anywhere; a missing `tagged_on` buckets as stalled plus a flag).

## G2: Backlog-review outcomes

If you run a periodic save-or-die backlog review posted in the team channel (items listed with
number reactions to "save" them), it is the agenda's idea source. Read the channel (last ~10
days), find the most recent post whose header matches your review's title pattern, and read its
number-emoji reactions plus its thread.

Classify per item:

- `saves` = who reacted with the item's number emoji (resolve Slack IDs to names via the
  roster).
- A thread reply that clearly asks to cancel/kill/drop an item is a **kill vote**; every other
  reply, ambiguity included, counts as a comment. A kill reply never counts as a comment.

Bucket each item: **interest** (at least one save or non-kill comment: a pull-candidate for
planning; a save beats a kill, so an interest item with kill votes is annotated "contested"),
**kill_voted** (kill votes only: headed for cancellation, never an adopt-candidate), or
**quiet** (no signal: on track for a cancellation proposal). Only interest items seed proposed
topics.

**No post found** (the review hasn't fired, or was skipped): omit the section and note the skip
in the run output; never fabricate signal. This source stays **read-only**: recording verdicts
on the Jira issues is the backlog review's job, not the agenda's.

## G3: People & availability

**Presence check first**: verify the who's-out query from `hr_availability_source` in
TEAM_CONTEXT.md is available (e.g. `command -v` on a CLI). If the field is absent, the tool is
missing, or it errors for missing credentials, **skip this source**,
note the skip and how to enable it, and continue; the agenda's other sections still stand. Never
fabricate availability.

Otherwise query who's out for the planning week (Monday through Sunday) however
`hr_availability_source` says to: an HR-system API, an export, or a shared leave calendar. If the
query runs through a CLI that mixes output streams, parse stdout only, since config notices may
print on stderr. Collect the absence entries (name, start, end) and compare
against the roster to compute:

- `out`: absences overlapping the week
- `out_during_slot`: absences covering the planning slot (flag these: the person can't attend)
- `in`: roster minus out
- `stalled_owner_out`: a stalled focus item whose assignee is out, the agenda's headline
  cross-flag

The HR read is strictly read-only; never book or modify time off.

## G4: WIP/flow trend *(optional)*

Read the WIP metrics page (`wip_metrics_page`), take the latest row's top oldest offenders, and
feed them as standing *Proposed topics* tying the agenda to the WIP-reduction goal. Skip if the
page read fails; it's optional.

## G5: Migrations needing attention

If you keep a cross-team migrations register page (`migrations_register_page`), read it, parse
the table rows (typically: Migration · Owner · Stage · Deadlines · Progress · Last comms · Next
nudge due · Status · Epic), and take the **active** rows (Status not Done).

Keep the rows that are **Stalled**, **At risk**, or **nudge-due** (next-nudge-due on or before
the week's Sunday), ordered by severity (stalled, then at risk, then nudge due). Done migrations
are dropped; an unparseable due date is flagged.

This is a **register read, not a live rollup**: it reflects whatever your migration-tracking
routine last wrote into the Status cells. If the register looks stale, that's a signal to run
that routine, not to recompute here. Read-only over the register and every other team's work.

## G6: Step-back schedule

> **Run this right after G0**: it decides whether the rest of the gather is even needed.

Every ~3 months the planning slot becomes the **quarterly step-back** (a review of the long-term
direction board, e.g. a Miro board, instead of weekly grooming). This source reads the schedule:

1. Read the quarterly step-back page (`stepback_page`).
2. Parse: the `Next step-back` date and the board link from the session-config table, the
   session-instructions text verbatim, and the **first data row** of the History table as the
   last entry (date, verdict, verdict notes, decisions, actions). A placeholder row or no table
   means no prior session (the first-session case). Pass entries through **verbatim**: a
   hand-edited verdict is displayed, never validated or rewritten on read.
3. **Short-circuit rule.** If the next-step-back date is on or before the week's Sunday, this is
   a step-back week: **skip G1, G2, G4, and G5 entirely** (the message replaces the weekly
   agenda and those reads would be wasted). **Still run G3** (the In/Out line is kept).
4. **Static link only.** The board is a link the humans open together; never drive the board
   tool's API from this skill.

Failure semantics are **asymmetric** (deliberate): no `stepback_page` field means the feature
isn't bootstrapped, so proceed in normal mode and say so in the run output. The field present
but the page unreadable is a configuration-load failure: hard stop, like every other configured
page.

## G7: Direction banner

Every weekly agenda **repeats the team's main direction**: everything the team does should point
at the current direction, so the weekly message re-states it each week rather than assuming
everyone remembers.

1. Read the direction page (`direction_page`).
2. Compose a **one-line summary of the page's current emphasis**, fresh from the page each run:
   the page is canonical and the banner inherits its edits. When the page carries a dated
   current-emphasis entry, summarise that; otherwise summarise the direction themes themselves.
   One line plus the page link; the banner repeats the direction, it never restates the page.

**Skip semantics:** on a step-back week, skip this read too (the step-back message *is* the
direction review). If TEAM_CONTEXT.md lists no direction page, or the read fails, omit the
banner and flag it in the run output; this is a soft dependency like G3, and the banner is never
fabricated from memory.

## G8: Compute

Assemble G1-G7 and bucket deterministically, identically every week (helper script recommended;
see the README caveats); pin "today" for reproducible runs:

- `week` (Monday-Sunday window) and `mode` (`normal` / `preannounce` / `stepback`, off the
  step-back next-due date: due this week or overdue = stepback; due next week = preannounce).
- `focus_review`: closed / advanced / stalled per G1's rules.
- `backlog_review`: interest / kill_voted / quiet per G2's rules.
- `availability`: out / out_during_slot / in / stalled_owner_out per G3.
- `migrations_needing_attention` per G5, ordered by severity.
- `proposed_topics`, ordered: stalled focus items first, then owner-out flags, then migrations,
  then backlog interest, then oldest WIP.
- `flags`: anything skipped, unparseable, or unmatched.

Hand the result to [agenda-message.md](agenda-message.md) to compose (dispatched on `mode`) and
post.

## Notes & edge cases

- **Run order matters.** The clean path is agenda **before** `planning-processor`. The G1
  fallback covers the late-run case but is lossy (no live label set); prefer the order.
- **Unmatched names** between the HR system and the roster surface as flags; reconcile by hand
  (a nickname, a new joiner) rather than dropping the person silently.
- **A past step-back date is the nag, not an error.** If `stepback-processor` never ran after a
  session, the date stays behind and every agenda stays in step-back mode with a growing overdue
  count. That pressure is the design. Never edit the date from this skill.
- **Midweek runs behave as today:** the mode keys off the week containing the slot, so a
  Wednesday run in the step-back week still renders step-back mode. Conversely, once the
  processor bumps the date on the step-back planning day, a same-week re-run renders normal mode with
  no pre-announce; harmless, don't "fix" it.
- **Planning-comment continuity holds across step-back weeks:** `planning-processor` still runs
  that planning day (invoked by `stepback-processor`), so the following week's `tagged_on`
  reconstruction works unchanged.
- **Everything here is a read.** If you find yourself about to write to Jira, Confluence, or the
  HR system, stop: the only write this skill makes is the Slack post.
