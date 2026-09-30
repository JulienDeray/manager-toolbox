---
name: backlog-resurfacing
description: |
  Runs a weekly save-or-die Backlog review: records last week's votes on the Jira issues, then posts
  the next 5 oldest Backlog items to the team channel (one vote saves an item for 6 months; silence
  or a "cancel" reply proposes cancellation). It writes comments, due dates, and the post without a
  gate, but cancels only in-session after the manager approves a full preview.
  Use when running the weekly backlog resurfacing, executing the pending cancel queue, or when the
  user mentions the save-or-die review or asks which backlog items are up for cancellation.
---

# Backlog Resurfacing (save-or-die)

A weekly review that keeps a large Backlog honest without a giant grooming session. The model: ideas
and undecided work are ordinary Backlog issues (no "parked" label, no separate parking-lot page, no
extra workflow status); everything in the Backlog is undecided by default, and the real decision
happens at the commitment line (Backlog to Next), not at filing time. What keeps the pool honest is
this review: every week 5 items are posted to the team channel; an item somebody saves (one vote is
enough) stays for another 6 months; an item nobody saves is proposed for cancellation, and a
cancellation is only ever executed with your explicit in-session approval. Silence kills, slowly and
reviewably.

This is a Jira + Slack routine that runs **fire-without-gate**: the weekly cycle gathers, writes its
factual Jira records, posts, and verifies in one pass, with no plan mode and no draft. The one hard
exception is the kill: the unattended run only ever proposes a cancellation; transitions to Cancelled
happen exclusively in the interactive `cancel` mode after you approve the previewed batch. Never
schedule messages via the Slack API (scheduled messages cannot be reviewed or cancelled from the
client); never deliver the weekly post as a draft (the post is the deliverable).

## Modes

- default: the weekly cycle (verdicts, selection, post, verify).
- `dry-run`: compose the verdicts and the post without any Jira or Slack write.
- `cancel`: the interactive pending-cancellation execution (manager approval + full-tree preview).

**First adoption:** run `dry-run` for a week or two first. The default mode writes Jira comments and
due dates and posts to the team channel without asking; it never cancels anything.

## Requirements

- **Jira MCP tools**: get issue, edit issue (due dates), add comment, get/apply transitions (cancel
  mode only).
- For every bulk JQL (the pool queries, the cancel-queue query): a paginated Jira REST search. MCP
  search tools can silently truncate result sets, and a truncated pool query silently shrinks the
  wheel (helper script recommended; see the README caveats).
- **Slack MCP tools**: read channel (find last week's post and its reactions), read thread, send
  message (the weekly post is the only Slack write).
- Optionally, a desktop-notification mechanism (e.g. `osascript` on macOS) as the unattended run's
  visibility surface.

If a required tool is missing or unauthenticated, including in the scheduled unattended run, say so,
stop, and post nothing. If reactions are not readable through your Slack tools, do not guess verdicts:
flag it, skip the verdict step, and say so in the run output (items simply stay in the wheel).

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root:

- `backlog_project`: the project whose Backlog is the pool.
- `team_channel`: where the weekly post goes.
- `manager_slack_id`: tagged in the post as the person who confirms cancellations.
- `team_roster`: Slack ID to display name, to name who saved what.
- `jira_site_url`: for browse links.
- `resurfacing_exclusions` (optional): labels or filters marking items this review must skip (e.g.
  items owned by another team's process).

If context loading fails, stop and say which field is missing. Do not proceed partially.

## The contract (locked rules)

- **Due dates on Backlog issues are the resurfacing clock, nothing else.** Empty due date = never
  reviewed (eligible once the item is older than the 3-month age floor). A set due date = the next
  scheduled review (stamped +6 months when an item is surfaced or saved). Do not use Backlog due
  dates for anything else; a real delivery deadline belongs on committed work.
- **One save vote suffices.** The default is death; a save is a cheap veto anyone can cast.
- **Saves expire.** A save buys about 6 months, then the item re-enters the wheel.
- **A kill comment is a cancellation vote, not interest.** A thread reply that explicitly asks to
  cancel, kill, or drop an item ("can be cancelled, stale") routes it to the pending cancel queue as
  pre-approved: the vote carries the same weight as your confirmation at the cancel review, so nobody
  re-judges it there. Anyone may cast a kill vote, exactly as anyone may save.
- **Ambiguity defaults to save.** Only clearly expressed cancel intent is a kill vote; any unclear or
  mixed reply counts as interest. A wrong save costs 6 months; a wrong kill costs a ticket.
- **Save beats kill.** An item with both a save signal and a kill comment is saved; surface the
  conflict in the run output so the kill voter can raise it at planning. The cheap veto stays cheap.
- **Kills never transition unattended.** The weekly run only writes proposal comments; the transition
  to Cancelled happens exclusively in-session at the cancel review, with you present.
- **Cascade on cancel.** Cancelling an Epic (or a Task with sub-tasks) includes its open children;
  the approval preview always shows the full tree.
- **Interest goes to planning.** Items with adoption or discussion interest become pull-candidates
  for the next planning session; the commitment decision stays in that ritual. A kill-voted item is
  never a pull-candidate.
- Both proposal comments contain the literal phrase `proposed for cancellation`: that phrase is how
  the cancel mode's queue query finds pending items, so any new proposal flavour must keep it.

### The `[RESURFACING]` comment vocabulary

The review trail lives on the issue as comments, all prefixed `[RESURFACING <date>]`:

| Comment | Written when | By |
|---------|--------------|-----|
| `Surfaced for save-or-die review: <post permalink>` | the item appears in a weekly post | the weekly run (factual) |
| `Saved by <names> (<reactions / comment>). Next review ~<date>.` | it got a save vote or interest comment | the weekly run (factual) |
| `No saves; proposed for cancellation, awaiting manager confirmation.` | a week passed with no signal | the weekly run (factual) |
| `Kill-voted by <names> ("<short quote>"); proposed for cancellation, pre-approved for the next cancel session.` | an explicit cancel reply, and nothing saved it | the weekly run (factual) |
| `Cancelled at the <date> review (<no saves / kill-voted by <names>>), manager-approved. <permalink>` | the kill was executed | cancel mode (in-session only) |

## Capability 1: the weekly cycle

One pass: verdicts, selection, post, verify.

### W1: Find last week's post

Read `team_channel`, last ~10 days. The resurfacing post is identified by its header text
`Backlog resurfacing, <date>`: match the text, not the markup (a channel read returns the bold as
`*single asterisks*`, whatever the post was written with). Take the most recent one; capture its permalink, its numbered
items (the issue keys), its reactions (which number emoji, from whom), and its thread replies.

- No post found (first run or a skipped week): skip W2 entirely, note it in the run output.
- Reactions unreadable: do not infer silence. Flag it, skip W2, say so in the run output. Items stay
  in the wheel (their due dates were already stamped when posted) and get re-picked when due again.
- **Double-run guard:** if the most recent post is under 5 days old, skip W2 (too early to judge) and
  do not post a second batch; report instead. `dry-run` is exempt.

### W2: Verdicts on last week's items

First classify every thread reply that is about a posted item as one of two kinds:

- **Kill**: the reply clearly asks to cancel/kill/drop the item. Only explicit cancel intent
  qualifies.
- **Interest**: everything else, ambiguity included (adopt, volunteer, discuss, a bare "keep", or a
  reply whose intent is unclear).

Then, per numbered item in that post:

1. **Saved** (its number emoji has at least one reaction, or it has an interest reply): add the
   `Saved by <names>` comment and re-stamp due = today + 6 months (a save buys 6 months from the
   save, not from the posting). Resolve reactor Slack IDs to display names via `team_roster`
   (unknown ID: keep the raw mention; some Slack tools return reactor identities in a form you cannot
   resolve, which is a flag, not a guess). A save signal beats any kill reply on the same item; note
   the conflict ("saved, contested by <kill voter>").
2. **Interest** (an adopting or discussing reply, more than a bare "keep"): also saved; additionally
   list it under pull-candidates in the run output with the reply quoted, for the next planning
   session.
3. **Kill-voted** (at least one kill reply, no save signal): add the kill-voted comment and add the
   item to the pending cancel queue flagged pre-approved. No due-date change (the posting stamp
   already keeps it out of the wheel while pending). No transition happens here.
4. **Unsaved** (no reaction, no reply): add the `No saves; proposed for cancellation` comment and add
   it to the pending cancel queue (awaiting the manager). No transition happens here.

Writes are sequential (parallel edit responses can shuffle) and each is verify-read (the comment
exists, the due date landed) before the run claims it.

### W3: Select this week's batch

Pool definition: the Backlog of `backlog_project`; issue types Task, Story, Bug, Epic (sub-tasks ride
with their parents); minus `resurfacing_exclusions`. Eligibility: a scheduled review has come due
(`due <= today`), or never reviewed and past the age floor (`due is EMPTY AND created <= -90d`).
Two JQL slices define the eligible set:

```
project = <backlog_project> AND status = Backlog AND issuetype in (Task, Story, Bug, Epic)
  AND due <= endOfDay()                          -- slice 1: scheduled reviews now due
project = <backlog_project> AND status = Backlog AND issuetype in (Task, Story, Bug, Epic)
  AND due is EMPTY AND created <= -90d           -- slice 2: never reviewed, past the age floor
```

Run both through a paginated search (truncation silently shrinks the wheel), merge, and **rank oldest
created first across the whole eligible set**: the wheel prioritises rot; a due date grants
eligibility, never a queue-jump. Take the first 5. Fewer than 5 eligible: post what there is. Zero
eligible: post nothing, log "wheel empty", and stop (still do W2's verdicts first).

**A child whose parent sits in the pending queue is skipped by the weekly selection**: the parent's
cascade can kill it anyway, and a save on the child would silently conflict with a kill on the
parent. The blast radius was already disclosed when the parent was posted via its
`(epic, N open children)` marker. Note the skip in the run output so you see it at the cancel review.

One rollout note: before the first run, clear any stale pre-existing due dates on Backlog items (old
delivery dates from years past), so slice 1 only ever contains wheel-stamped or deliberately seeded
review dates.

### W4: Compose + post

One message to `team_channel`. Follow the Slack message formatting conventions in `TEAM_CONTEXT.md`.
Specific to this post: the items are numbered `1.` lines, each separated by a blank line
(intentional: each renders as its own paragraph); issue keys are markdown links to
`<jira_site_url>/browse/<KEY>`; keep ontology jargon out of the prose (render a label like
`axis::reliability` as "(reliability)"). Shape:

```
**Backlog resurfacing, <date>**

These 5 have been rotting in the Backlog. Which do we keep?

React with an item's number to *save* it (one vote is enough). Comment to adopt or discuss.
Comment "cancel" on an item to vote it out (a save still beats a kill vote).
No reaction = proposed for cancellation next week (<@manager_slack_id> confirms before anything is cancelled).

1. [PROJ-101](...) · 1930 days · auto-archive stale feature flags

2. [PROJ-142](...) · 1470 days · alert on failed nightly backups (reliability)

...

Last week: PROJ-231 saved (Alex), PROJ-118 proposed for cancellation.
```

- Age = days since created ("rotting" is the deliberate framing; keep it).
- The last-week footer is one line summarising W2's verdicts; omit it when W2 was skipped. It
  distinguishes the outcomes: `PROJ-137 kill-voted (Sam)` vs `PROJ-118 proposed for cancellation`
  (silence) vs a contested save (`PROJ-140 saved (Alex), contested by Sam`).
- An Epic gets an `(epic, N open children)` marker so voters know the blast radius.

### W5: Record the surfacing

For each posted item, sequentially: add the `Surfaced for save-or-die review: <permalink>` comment
and stamp due = today + 6 months (this removes it from the eligible pool while its verdict is
pending). Verify-read each.

### W6: Verify + surface the run output

1. Verify-read the channel: the post landed; capture the permalink.
2. Run output: the permalink, the batch, W2's verdicts (with any save-vs-kill conflicts called out),
   and the full pending cancel queue (this week's additions plus anything still unresolved), split
   into pre-approved (kill-voted) and awaiting-manager (silence) entries.
3. Fire a desktop notification whenever the cancel queue is non-empty or something failed, e.g.
   "N items pending cancellation (K pre-approved): run /backlog-resurfacing cancel". Re-notify on
   every subsequent run until the queue is resolved. Nothing pending and nothing failed: silent
   success, one-line log.

## Capability 2: cancel execution (`cancel`, in-session only)

Never runs unattended. If this mode is somehow reached in a scheduled run, stop before the preview:
it requires a human answer, and a synthetic approval is not one. In session it collects the pending
queue from the issues' own `[RESURFACING]` comments (C1), fetches every candidate's child tree (C2),
presents one preview table where awaiting-manager rows need your call and pre-approved (kill-voted)
rows execute unless you veto them (C3), cancels the approved items children-first with the
resolution set explicitly (C4), and verify-reads everything (C-verify).

**Full procedure:** [references/cancel-mode.md](references/cancel-mode.md).

## Unattended-write boundary (locked)

The scheduled run's Jira writes are factual records only: `[RESURFACING]` comments and due-date
stamps. It never transitions an issue, never edits summaries, descriptions, or labels, never posts
anywhere but the one weekly channel message. The kill always goes through the manager.

## Scheduling

The weekly cadence is a scheduled task in your Claude Code harness (pick a quiet slot clear of your
other ritual traffic; mid-week mid-morning works well). The task's prompt simply invokes
`/backlog-resurfacing`; scheduling machinery stays out of the skill. A missed slot is fine: everything
keys off "today", not the nominal weekday. Note that Jira and Slack tools authenticated interactively
can be unavailable to a headless run; the run must then stop and say so rather than posting partial
output.

## Usage

```
/backlog-resurfacing            # the weekly cycle (what the scheduled task runs)
/backlog-resurfacing dry-run    # see what it would do without writing
/backlog-resurfacing cancel     # execute pending cancellations (interactive, previews + approval)
```
