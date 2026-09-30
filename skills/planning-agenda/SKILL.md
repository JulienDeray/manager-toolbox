---
name: planning-agenda
description: |
  Builds the team's weekly planning agenda: the pre-planning briefing posted ~40 minutes before
  the weekly grooming/planning session, so it starts informed instead of cold. The before-bookend
  sibling of planning-processor. Fire-without-gate: one of the two direct-post skills, it posts
  the agenda straight to the team channel with no approval step. Once a quarter it switches to
  step-back mode.
  Use when preparing the weekly planning or grooming session, building the pre-planning agenda
  or briefing, or when the user asks who's out this week or what to put on the planning agenda.
---

# Weekly Planning Agenda

The **before-bookend** of the team's weekly planning/grooming ritual. It runs ~40 minutes before
the session and posts the agenda, so the ritual opens on the right topics instead of cold. It is
the sibling of the `planning-processor` skill (the *after* bookend) and reuses the same Step-0
config load, but **deliberately not** its plan-mode gate: this is a **fire-without-gate** skill,
one of the toolbox's two direct-post skills (the other is `backlog-resurfacing`). It gathers,
composes, and posts in one go.

> **Why fire-without-gate:** the agenda is low-stakes, formulaic, and scheduled (it runs
> unattended on a weekly schedule with nobody around to approve a draft). A draft nobody
> sends is worse than a post. One rule applies regardless: **never use
> Slack's schedule-message API**. API-scheduled messages do not appear in Slack's "Drafts &
> sent" UI, so they are invisible, uneditable, and uncancellable from the client until they
> fire. Timing lives in the harness's scheduled *task* (which runs the skill, which posts
> immediately); a message is never left pending inside Slack.

This is a **Jira + Confluence + HR + Slack** routine, but it is **read-only over Jira,
Confluence, and HR data**: it never edits a ticket, a page, or HR records. Its **only write is
one Slack post** to the team channel, posted directly, with no draft and no approval step.

## Requirements

- **Jira MCP tools** (read-only here): get issue with changelog, plus JQL search. Run the bulk
  JQL searches through the Jira REST API rather than the MCP search tool: MCP search results
  silently truncate, and this skill runs unattended, so a short result set has nobody to notice
  it (helper script recommended; see the README caveats). If the REST path is unavailable, fall back to per-status MCP slices and say so in the posted
  agenda.
- **Confluence MCP tools** (read-only): get page.
- **Slack MCP tools**: the send tool (the post, the only write), plus channel/thread reads for
  the backlog-review source and the post-fire verify. **Forbidden in this skill:** the
  schedule-message tool (see above) and the draft tool (a weekly agenda nobody sends is worse
  than a post).
- **A who's-out source** (an HR-system API, an export, or a shared leave calendar) for the availability
  section. This is the one **soft dependency**: if it's absent or unauthenticated, degrade
  gracefully, omit the In/Out section, flag that it was skipped and how to enable it, and still
  produce the rest of the agenda. Never invent availability data.

If a **required** tool (Jira, Confluence, or Slack) is missing or unauthenticated, including in
a scheduled unattended run, **say so, stop, and post nothing**; never queue, retry against a
different surface, or leave a partial message.

## Step 0: load team context

Read **`TEAM_CONTEXT.md`** at the repo root. This skill needs:

- `jira_project_keys`, `jira_site_url`, and `epic_board_id` (the epic planning board)
- `team_channel` and `team_roster` (name to Slack ID; the roster also feeds the *In this week*
  list and the HR name match)
- `planning_slot` (e.g. Mon 10:00) and `team_timezone`
- `planning_comment_prefix` (e.g. `[PLANNING DD-MM-YYYY]`) and `date_format`
- `team_ontology_page` (focus-label rules) and `team_agreement_page` (WIP context), if kept
- `wip_metrics_page` (optional; feeds the WIP-trend topics)
- `migrations_register_page` (optional; feeds the migrations section, skip if absent)
- `stepback_page` (the quarterly step-back page; see the asymmetric failure rule below)
- `direction_page` (optional; feeds the direction banner)
- `hr_availability_source` (optional; the who's-out query for the availability section)

**Asymmetric failure for the step-back page (deliberate):** if the field is **absent** (the
feature isn't bootstrapped), soft-degrade: proceed in normal mode and flag "step-back not
configured" in the run output. If the field exists but the **page can't be read**, that is a
configuration-load failure like any other: hard stop. Any missing required field is likewise a
hard stop; say which one and do not proceed with partial configuration.

## Flow: gather, compose, post, verify

1. **Gather + analyse (read-only):** pull the context sources
   ([references/gather-context.md](references/gather-context.md)), compute the agenda buckets
   and mode, and compose the message.
2. **Post + verify:** post with the Slack send tool, verify-read the channel, and surface the
   permalink plus any flags in the run output.

A `dry-run` argument (or an explicit "just show me the agenda") composes and prints the message
**without posting**: the only path that skips the write.

## Context it gathers

Each source is read-only; full procedures in
[references/gather-context.md](references/gather-context.md).

1. **Last week's focus review**: the items that carried the focus label (e.g. `prio::now`),
   bucketed **closed / advanced / stalled** since they were tagged. Stalled items are the first
   agenda candidates, and a stalled verdict is only reached after a **deep check**: an epic's
   own status rarely moves, so movement in its children and description edits count as
   advancement. *Ordering hazard:* `planning-processor` clears-and-reapplies the focus label
   weekly, so read this snapshot **before** the processor clears it (the natural before/after
   order); if it was already cleared, reconstruct last week's set from the most recent prior
   planning comments. Never modify `planning-processor` to work around this.
2. **Backlog-review outcomes**: the latest backlog-resurfacing post in the team channel (if you
   run a periodic save-or-die backlog review): items with saves or comments become
   **pull-candidates** for planning; quiet items are on track for a cancellation proposal. Skip
   the source if you don't run one.
3. **People & availability**: who's out this week (especially during the planning slot, or
   holding a stalled focus item) and who's in, from your HR system. Degrades gracefully.
4. *(optional)* **WIP/flow trend**: the top oldest offenders from the WIP metrics page, as
   standing agenda candidates tying the agenda to the WIP-reduction goal.
5. **Migrations needing attention**: a read of the cross-team migrations register (if you keep
   one) for active migrations that are stalled, at risk, or nudge-due. This is a **register
   read, not a live rollup**: it trusts the register's Status cells, which a separate tracking
   routine keeps fresh. If the register looks stale, run that routine; don't recompute here.
6. **Step-back schedule**: a read of the quarterly step-back page (next-due date, board link,
   session instructions, and the newest history row = "what was discussed last time"). Read
   this **right after determining the planning week**: a step-back week short-circuits sources
   1, 2, 4, and 5 entirely.
7. **Direction banner**: a read of the direction page so every weekly agenda **repeats the
   team's main direction**: everything the team does should point at the current direction, so
   the weekly message re-states it rather than assuming everyone remembers. One line plus the
   page link, composed fresh from the page each run; skipped on step-back weeks (that message
   *is* the direction review) and omitted-with-a-flag if the page can't be read. Never write it
   from memory.

## Output: the agenda message

A terse agenda message, posted directly to the team channel ~40 minutes before planning. The
gather determines a **mode** that picks the shape:

- **`normal`** (the usual week): direction banner, then sections: *In / Out this week* ·
  *Carried-over focus* (closed / advanced / stalled) · *Backlog review* (interest =
  pull-candidates; quiet count) · *Migrations needing attention* · *Proposed topics*.
- **`preannounce`** (the step-back date falls in the following week): the normal agenda plus a
  final one-line heads-up with the long-term board link.
- **`stepback`** (the step-back date is in this week, or already past): the message **replaces**
  the weekly agenda: availability line kept, then the step-back banner, the board link, the
  page's session instructions, and last quarter's verdict and decisions. No focus / backlog /
  migrations sections that week.

**Self-healing rule:** a next-step-back date left in the past keeps the agenda in step-back mode
every week (with a growing overdue count) until the `stepback-processor` skill bumps it. That
nag is intentional: it is the recovery signal for a forgotten wrap-up. Never "fix" the date from
this skill; it stays read-only over Confluence.

The message uses terse style: drop filler, but numbers, all ticket keys, `<@mentions>`, and
links stay verbatim.

**Full procedure:** [references/agenda-message.md](references/agenda-message.md): the
section-by-section shape, formatting rules, timing, the post call, and the post-fire verify.

## Scheduling

The weekly run is a **scheduled task in the operator's agent harness** (e.g. 09:20, 40 minutes
before a 10:00 planning slot, on the `planning_slot` weekday), not part of this skill: the task's prompt simply invokes
`/planning-agenda`. To change the cadence, edit the scheduled task, not this file. Scheduling
machinery stays out of skills.

## Usage

```
/planning-agenda                        (default slot from TEAM_CONTEXT.md)
/planning-agenda <YYYY-MM-DDTHH:MM>     (one-off reschedule)
/planning-agenda dry-run                (compose without posting)
```
