# Slack wrap-up + WIP/flow metrics: detailed procedure

> Capability 2 of the `planning-processor` skill. Computes the weekly **WIP snapshot**, appends
> a row to the WIP metrics Confluence page (`wip_metrics_page` in TEAM_CONTEXT.md), and drafts a
> terse Slack wrap-up for the team channel. Implements the WIP-reduction meta-goal of the weekly
> routine. Examples use the invented project keys `PROJ` (tracked project) and `OPS` (secondary
> project sharing the epic board).

## Contents

- Locked definitions
- Inputs
- M1: Query in-progress epics (read-only)
- M2: Compute age per epic (changelog)
- M3: Aggregate
- M3b: Board lane counts (optional, read-only)
- M4: Append the metrics row (Confluence)
- M5: Deliver the Slack wrap-up as a channel draft
- M6: Refresh the ontology's active-epics list (optional)
- M-verify (mandatory after writes)
- Notes & edge cases

Every write happens plan-mode, approve, execute, verify. The Confluence row and the Slack
message are both writes; nothing lands before the plan is approved.

## Locked definitions (do not silently change: the metric is tracked over time)

- **WIP = in-progress epics in the tracked project only.** Ticket-level (delivery-board) WIP is
  out of scope for the count; this matches the meta-goal of reducing in-progress *epics*. If the
  epic board also shows a secondary project's epics, those are deliberately **not** in this
  count (a locked series must not change definition mid-stream); the lane counts (M3b) are where
  the board's full population is reported, so the board-vs-message gap is explained in the same
  breath.
- **"In progress" = active columns only**: e.g. statuses `In Progress` + `To review`. A
  committed-but-not-started column (e.g. `Next`) is **excluded**, as are Backlog and Done.
- **Age = days since the epic entered the active set**, from the changelog (M2). Not since
  created.
- **Lane counts (optional) = the epic board's population bucketed by swimlane**, per your
  board's lane model (e.g. Team work / Long running / Solo work, keyed off labels). The set is
  non-Backlog, non-Done epics across the board's projects: broader than WIP, narrower than
  everything. If your team drives one lane's share (e.g. "grow the Team-work share"), track that
  share number week over week against its baseline.

## Inputs

None required beyond Step-0 config. **Optionally**, if Capability 1 (priority highlighting) ran
in the same session, reuse this week's focus set to headline the wrap-up's *This week's focus*
line. If it didn't run, omit that line rather than re-querying.

## M1: Query in-progress epics (read-only)

```
project = PROJ AND issuetype = Epic AND status in ("In Progress", "To review") ORDER BY key
```

**Run it via the Jira REST API, not the MCP search tool**: MCP search results truncate silently
(a hard row cap on some implementations), and this query returns 15-30 epics, so on the capped
path the WIP count and the total-age figure would both be quietly computed from a fraction of
the board (helper script recommended; see the README caveats).

Robustness: prefer **named statuses** over `statusCategory`. `statusCategory = "In Progress"`
alone wrongly includes a committed-but-not-started column like Next (Jira puts it in the same
category). If a workflow rename ever breaks the named statuses, fall back to
`statusCategory = "In Progress" AND status != "Next"`, and fix the names.

Keep the requested field set **minimal** (`key,summary,status,assignee,labels,
statuscategorychangedate`): a full epic payload is large and can blow response limits.
Sanity-check the returned count against last week's WIP number on the metrics page before
computing deltas: a sudden halving is a truncation smell, not a real drop, and a sudden doubling
smells like accidental scope widening (the WIP count is single-project by definition), not real
growth.

## M2: Compute age per epic (changelog)

For each epic, fetch the issue with its changelog expanded and a minimal field list. In the
changelog histories, the relevant items are status changes with from/to values.

**The active-entry rule:** age = days since the epic most recently **entered** the active set
from **outside** it: the most recent transition whose `to` status is in {In Progress, To review}
**and** whose `from` status is not. Transitions *within* the active set (In Progress to To
review and back) do **not** reset age; a move out to Next/Backlog and back **does** (the epic is
actively worked again). Age counts from that date, not from created.

This is mechanical, so every week should be computed identically (helper script recommended; see
the README caveats); if walking changelogs by hand, apply the
rule exactly and watch these edge cases:

- **No qualifying transition** (created directly into In Progress, or an empty changelog): fall
  back to the `statuscategorychangedate` field, mark the age **approximate**, and flag it. The
  fallback can over-count because a Next-style column shares the In-Progress status category;
  surface the flag in the metrics row's Notes.
- **Paginated changelog:** if the changelog's total exceeds the histories returned and no
  qualifying transition was in the page, page the changelog and re-check before trusting the
  fallback.
- **Migrated epics** (keys moved between projects) keep their pre-migration status history, so
  the age is correct; there is no project-change item in the status walk to worry about.

## M3: Aggregate

- **Epic WIP** = the M1 count.
- **Total age** = sum of ages, in epic-days.
- **Avg age** (1 decimal).
- **Top 5 oldest**, each as `KEY · Nd · summary`. These are the close-or-split candidates: the
  headline of the WIP-reduction story.
- **Delta vs last week** = compare WIP and total age against the previous row on the metrics
  page (M4 reads it). First run has no previous row: report absolute values only. Track how many
  ages were approximate fallbacks.

## M3b: Board lane counts (optional, read-only)

If your epic board uses label-keyed swimlanes, bucket the board's population by lane so the
lane mix is visible week over week:

1. **Tracked project set:** all non-Backlog, non-Done epics
   (`project = PROJ AND issuetype = Epic AND statusCategory != Done AND status != Backlog`).
   Reuse this payload for M6.
2. **Secondary project set:** the equivalent query per additional board project, mirroring the
   board's own filter exclusions. Gotcha: if a project has a literal **Backlog** status,
   `statusCategory != Done` does not exclude it; add `status != Backlog` explicitly, or the lane
   counts silently include the whole backlog.
3. **Bucket by the board's swimlane precedence, first match wins**, per the lane model on your
   ontology page (that page is authoritative over this worked example). Example model:
   `long` (carries `wt::long`: not planned weekly; wins when both labels present), `team`
   (carries `area::team`: 2+ people actively collaborating), `solo` (everything else, the
   default lane: one owner, no label needed).
4. Report the counts and the driven share, e.g. `team 5 · long 6 · solo 12 · team share 22%`,
   with a delta against the previous row's Lanes cell.

No script needed: a group-by over the label arrays. An epic with no lane label lands in the
default lane; that is a legitimate state, not a flag. Never invent a label to make the grouping
tidy. A default-lane epic the session described as a collaboration goes to the ticket-hygiene
hand-off (Capability 4) as a label proposal.

## M4: Append the metrics row (Confluence)

1. Read the metrics page body **and its version number**. Grab the previous row for M3's deltas.
2. **If the page is missing**: create it first with the header schema below, then proceed; keeps
   the routine idempotent.
3. **Splice the new row** into the table, newest-at-top (directly after the header row), leaving
   every other byte of the body untouched. This is the fragile part (easy to clobber a row when
   hand-editing structured Confluence HTML); do a pure-string table splice (helper script
   recommended; see the README caveats). If editing by hand,
   diff the new body against the old and confirm the only change is the inserted row.
4. Update the page with `version = previous + 1` and a short version message
   (`Weekly WIP row <date>`).

**Row columns** (must match the page's table header exactly):

| Week | Epic WIP | Total age (epic-days) | Avg age (d) | Top 5 oldest (KEY · age) | Lanes | Notes |

- **Week** = the planning date, in `date_format`.
- **Top 5 oldest** = `PROJ-101 · 58d` entries, comma- or line-break-separated.
- **Lanes** = the M3b counts (omit the column if you don't lane-count; older rows from before
  the column existed keep a `-` placeholder, no retro-computation).
- **Notes** = delta vs last week (e.g. `WIP -2, total age -40d`) plus any approximate-age flags
  from M2.

**Storage-format caveat:** editing a Confluence table means editing structured HTML. Round-trip
the body in the same format you fetched it, splice only the new row, and **verify-read**
(M-verify) before claiming success.

## M5: Deliver the Slack wrap-up as a channel draft

Draft a **terse** message for the team channel. Follow the Slack message formatting conventions
in `TEAM_CONTEXT.md` (mentions, links, bold, no em dashes). Use the **digest shape**: a handful
of short, self-contained standalone lines (a metric per line, no list markup), with **one blank line between every
line**, so Slack renders clean paragraphs. (This is the opposite of the bulleted list shape:
never mix them; a digest line never grows blank-separated sub-bullets.) **Every metric carries
its delta vs last week in parentheses** right after the number, so the trend reads inline:

```
**Weekly planning wrap-up: DD-MM-YYYY**

:dart: **Focus this week:** <focus items, if Capability 1 ran; else omit this line>

:bar_chart: In-progress epics: **14** (-2 vs last week)

:thread: Lanes: team **5** · long 6 · solo 12 · team share **22%** (+3)

:hourglass: Total age **312** epic-days (-40 vs last week) · avg **22.3d** (-1.1)

:hourglass_flowing_sand: Oldest (close/split candidates): [PROJ-187](<url>) · 58d, [PROJ-207](<url>) · 44d

:page_facing_up: [WIP & Flow Metrics](<metrics page link>)
```

First tracked week: no previous row, so omit the parentheses rather than writing "(n/a)". If the
previous week's numbers weren't comparable (approximate ages), keep the delta but note it in the
Confluence row, not the message. Keep it short and scannable: the team gets the trend in about
seven lines.

**Delivery = Slack draft in the channel.** Once the plan is approved, create the message as a
channel-attached draft via the draft tool, no extra confirm step: the manager edits it in Slack's
"Drafts & sent" and sends by hand. Never call the send tool for it. Slack allows one attached
draft per channel: if one already exists, report it and hand over the text instead. If the draft
tool is unavailable, the text in chat is the deliverable.

## M6: Refresh the ontology's active-epics list (optional)

If your team ontology page carries a living **active epics** list (all non-Backlog, non-Done
epics, grouped by lane), this weekly run is the mechanism that keeps it current. Note this is a
**broader set than the WIP count** (M1 covers the active columns only; the active list also
includes Next, On Hold, etc.).

1. Query the active-epics set (minimal fields, via the REST path as in M1: this set is larger
   than M1's, so silent truncation would drop epics from the page and the diff would read them
   as closed). Reuse M3b's payloads if it ran this session.
2. Read the ontology page; extract the current listed epic keys.
3. **Diff**: epics to add (newly active) and to drop (closed or moved back to Backlog). If the
   diff is empty, skip the write and note "active list unchanged".
4. If changed, regenerate **only** the active-epics section, grouped by the board's lane
   precedence, update its "as of <date>" line, and update the page with `version = previous +
   1`. Leave every other section byte-for-byte. An epic with no lane label lands in the default
   lane; that is legitimate, never invent a label.
5. The diff (added / dropped keys) is part of the approval plan like every other write.

## M-verify (mandatory after writes)

1. **Metrics page:** re-read; confirm the new row is present, the header and prior rows are
   intact, and the version incremented by exactly 1.
2. **Ontology page (if M6 wrote):** re-read; confirm the active-list section matches the
   approved diff, the other sections are untouched, version +1.
3. **Slack:** confirm the draft was created (the tool returns a draft id and channel link) and
   report both. The message itself is sent by the manager from Slack; never report it as "posted".
4. Report the reconciled result plainly: WIP count, ages, top offenders, deltas, approximate-age
   flags, and the active-list diff (or "unchanged"). Only claim success once the verify-reads
   match the intended writes.

## Notes & edge cases

- **WIP is single-project; lanes cover the whole board.** A stray key from an out-of-scope
  project in the notes is skipped and flagged. A secondary-project epic never enters the WIP
  count or the age math, only the lane counts and the active list.
- **Approximate ages.** The status-category-date fallback can over-count (a Next-style column
  shares the In-Progress category, so an epic that sat in Next shows the older entry date).
  Always prefer the changelog active-entry rule; flag any fallback rows.
- **Column WIP limits are not epic limits.** If your team agreement sets per-column ticket
  limits, don't apply them to the epic count; report the epic count and trend without inventing
  an epic cap.
- **First run**: no previous row, no deltas, and M4 may need to create the page.
- **Empty result** (0 in-progress epics) is still a valid row: write it; a zero is a meaningful
  trend point.
