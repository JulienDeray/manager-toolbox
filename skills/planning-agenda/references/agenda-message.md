# Agenda message: detailed procedure

> The write half of the `planning-agenda` skill: compose the agenda from the gathered context
> and **post it directly** to the team channel. No plan-mode preview, no draft, no approval
> (fire-without-gate is this skill's deliberate design). This is the skill's **only write**.

## Contents

- M1: Compose the message
- M1-stepback: the quarterly step-back variant
- M2: Timing
- M3: Post the message
- M-verify (mandatory after the write)
- Notes & edge cases

## M1: Compose the message

One terse agenda for the team channel (`team_channel`), built from the gathered context.
**Dispatch on the computed `mode` first:**

- **`stepback`**: the replacement template in *M1-stepback* below (no weekly sections).
- **`preannounce`**: the normal agenda, plus a one-line heads-up appended as the final line:

  ```
  :compass: Heads-up: next week's planning slot is the quarterly step-back (<date>). Board: <board-url>. Skim it before then.
  ```

- **`normal`**: the weekly template below, unchanged.

Weekly template, suggested shape (drop a section if it's empty, except keep the focus header so
"nothing stalled" stays visible). **List shape:** blank lines between sections only; lines within
a section are consecutive, never blank-separated (a blank line between list items makes Slack
render each as its own single-item list, doubling the vertical gap).

```
**Planning agenda: <slot date>** (planning <slot time>)

:compass: Direction: <one line from the direction page> ([Direction](<page-url>))

:beach_with_umbrella: Out this week: <@SLACK_ID> (06-08), <@SLACK_ID> (09-10)  ·  In: everyone else
:warning: Out during the slot: <@SLACK_ID>

**Carried-over focus**
:white_check_mark: Closed: PROJ-136
:arrow_forward: Advanced: PROJ-207
:no_entry: Stalled (discuss first): PROJ-187 (owner <@SLACK_ID> out this week) · PROJ-229

**Backlog review (from the <date> post)**
:seedling: Interest: [PROJ-243](...) connection pooling (saved by Alex): adopt/promote?
:x: Kill-voted: [PROJ-171](...) library-upgrade spike (Sam), joins the cancel queue pre-approved
:hourglass: Quiet so far: 3 items, on track for a cancellation proposal

**Migrations needing attention**
:red_circle: build-tool upgrade: stalled (owner <@SLACK_ID>) · PROJ-130
:large_yellow_circle: streaming-library migration: at risk · PROJ-123
:bell: secrets-manager rollout: nudge due 10-07 · PROJ-140

**Proposed topics**
- Unblock PROJ-187 (stalled, owner away)
- <migrations + backlog interest + oldest-WIP seeds>
```

Mapping from the gathered context:

- **Direction banner**: the G7 read. One line re-stating the team's current direction, fresh
  from the direction page each run, plus the page link. It's here every week because everything
  the team does should point at the current direction; the agenda repeats it rather than
  assuming everyone remembers. Omit the line (and flag it in the run output) when the page
  couldn't be read; never write it from memory. Not rendered in `stepback` mode (that message
  *is* the direction review).
- **Out / In this week** and **Out during the slot**: from G3's availability buckets.
- **Carried-over focus**: closed / advanced / stalled from G1, **stalled first and called
  out**, annotating any stalled item whose owner is out this week. Advanced items that moved
  **indirectly** quote the *how* from their evidence trail, e.g.
  `PROJ-204 artifact-registry migration (via subtasks: PROJ-218, PROJ-219 moved to Done)` or
  `(description: checklist 3 items ticked)`. Stalled items state the depth of the verdict: "no
  movement in status, children, or description since <tagged_on>" (or, if the deep check was
  flagged missing, say so: "status only, deeper check missing").
- **Backlog review**: one line per **interest** item (key + who saved / how many comments;
  these are the pull-candidates; a contested item appends "(contested: kill vote from <name>;
  save wins)"), one line per **kill-voted** item (key + who kill-voted, stated as headed for
  cancellation, never phrased as an adopt-candidate), one summary line for the **quiet** count.
  Section omitted when no review post was found (say so in the run output, not the message).
- **Migrations needing attention**: already ordered stalled, at risk, nudge due. One line per
  migration: the status and reason, owner (tagged via Slack ID), the epic key, and, for a
  nudge-due row, the due date. If the register is configured but empty of attention items, say
  "none".
- **Proposed topics**: the ordered seeds (stalled focus, owner-out, migrations, backlog
  interest, oldest WIP).

**Verbatim, always:** every ticket key, every `<@SLACK_ID>` (resolve names to Slack IDs from the
roster), every number and date, and every link. Follow the Slack message formatting conventions
in `TEAM_CONTEXT.md` (bold, links, no em dashes). This message uses the list shape: `-` bullets
under bold section headers, ticket keys as `[PROJ-101](<jira_site_url>/browse/PROJ-101)` links,
and no blank line between consecutive bullets.

### M1-stepback: the quarterly step-back message (replaces the weekly agenda)

Once a quarter the slot is the step-back session. The message **replaces** the weekly content;
availability is the only weekly section kept (no carried-focus, no backlog review, no
migrations, no proposed topics; their reads were skipped in the gather):

```
**Quarterly step-back: <slot date>** (planning <slot time>)

:beach_with_umbrella: Out this week: <@SLACK_ID> (...)  ·  In: everyone else
:warning: Out during the slot: <@SLACK_ID>

:compass: This week's planning slot is the *quarterly step-back*: we review the long-term board
together and check our day-to-day priorities against the direction.
Board: <board-url>
<session instructions, quoted verbatim from the step-back page, 1-3 lines>

**Last time (<last entry date>)**: <verdict>. <verdict notes> · <decisions one-liner>
```

- Insert an **overdue line** right after the banner only when the step-back is overdue:
  `It was due <date> (<N>d overdue): run it this week.`
- No prior session (first step-back): the last line becomes
  `First step-back, no prior session.`
- Board link missing from the page: do **not** fabricate one; surface the flag in the run
  output and leave the Board line out until the page is fixed.
- The In/Out lines follow the weekly rules (the HR-skip degrades the same way).

## M2: Timing

The agenda should land **before the slot**, resolved in `team_timezone` (so the wall-clock time
is correct across DST switches). The weekly cadence is a scheduled task in the operator's
harness (e.g. 09:20 for a 10:00 planning slot); the skill posts as soon as the message is
composed. A late run (inside the last minutes before the slot, or after it) still posts: a late
agenda beats no agenda; just say in the run output that it landed late. A run for a rescheduled
slot posts immediately too, whenever it is run.

> **Do not schedule the message.** API-scheduled Slack messages don't appear in Slack's "Drafts
> & sent" UI: invisible, uneditable, and uncancellable from the client until they fire. Timing
> lives in the harness's scheduled *task* (which runs the skill, which posts immediately), never
> in a message left pending inside Slack.

## M3: Post the message

Post directly to the team channel with the Slack send tool. No preview step, no draft, no
approval: the tool for this step is the send tool and nothing else; never the schedule tool, and
the draft tool is not used. The **only** no-post path is an explicit `dry-run`
(argument, or the operator asking to just see the agenda): then print the composed message, the
mode, and the flags, and stop.

## M-verify (mandatory after the write)

1. Confirm the post landed: verify-read the channel (or capture the returned permalink) and
   surface the link in the run output.
2. Report the **mode** (and, for `stepback`, that the weekly sections were deliberately skipped)
   and any **flags** from the gather (unmatched names, skipped HR read, no backlog post found,
   unparseable step-back date, missing board link). Flags annotate the post; they never
   retroactively unsend it. If one would have changed the content materially, say so and let the
   operator decide on a follow-up post.
3. Claim success only once the post is confirmed; if the send failed, say that and post nothing
   else.

## Notes & edge cases

- **Auth caveat.** The scheduled weekly run is unattended: if the Slack or Jira/Confluence MCP
  is missing or unauthenticated at run time, stop and post nothing; never queue, retry against a
  different surface, or leave a partial message.
- **The write is singular.** One post per run, to the team channel only. A re-run (e.g. after a
  config fix) means a second message in the channel: say so before re-posting in the same week,
  and prefer putting corrections in a thread reply on the original post.
- **Empty sections.** If nothing is stalled, nothing is due, and nobody is out, keep the headers
  terse and say so ("Stalled: none") rather than omitting silently; the absence is itself useful
  at planning.
