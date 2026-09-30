# Cancel execution: detailed procedure

> Capability 2 of the `backlog-resurfacing` skill: the interactive `cancel` mode that executes the
> pending cancellation queue after the manager approves a previewed batch. Examples use the invented
> project key `PROJ`.

## Capability 2: cancel execution (`cancel`, in-session only)

Never runs unattended. If this mode is somehow reached in a scheduled run, stop before the preview:
it requires a human answer, and a synthetic approval is not one (scheduled runs of approval-gated
flows produce a report, not writes).

### C1: Collect the pending queue

The queue is defined by the issues themselves, not session memory: an issue is pending when its
latest `[RESURFACING]` comment is a cancellation proposal (a later `Saved` or `Cancelled` comment
resolves it). Find candidates with:

```
project = <backlog_project> AND status = Backlog AND comment ~ "proposed for cancellation"
```

(through a paginated search), then read each candidate's comments and keep only those whose latest
`[RESURFACING]` entry is a proposal. Carry the flavour (silence vs kill-voted), the voter names, and
the quote forward. Empty queue: say so and stop. An already-moved candidate (no longer in Backlog)
resolved itself: drop it and note it.

### C2: Fetch the child trees

One JQL for all candidates: `parent in (<keys>)`, any status. Open children (statusCategory != Done)
are included in the cancel batch: a cancelled parent never leaves live orphans. Done or already
cancelled children are listed for context but never touched.

### C3: Present the preview (the gate the whole design hangs on)

One table for the whole batch: key, type, age, proposed-at date (with the post link), decision state,
children included, summary. Ask explicitly. Awaiting-manager rows need your call: approve, approve a
subset, or decline. Pre-approved rows (kill-voted) are presented as already-decided and execute
unless you veto them here (the veto is the only remaining out, which is why even pre-approved kills
never run unattended). A declined or vetoed item is a save: add a `Kept by manager decision at cancel
review` comment and stamp due = today + 6 months.

A child whose parent sits in the pending queue is skipped by the weekly selection (see W3 in
[SKILL.md](../SKILL.md#w3-select-this-weeks-batch)); the run output notes each skip so you see it here.

### C4: Execute (approved items only)

Sequentially per item, never in parallel:

1. Children first, then the parent: read the live transitions, find the one targeting Cancelled,
   apply it. Resolve the transition live; never assume an id.
2. Set the resolution explicitly (e.g. "Won't Do"). **Some Jira workflows do not set Resolution on
   transition** (a screenless transition leaves it empty, which reads as unresolved in every report):
   set it right after the transition, never assume the transition did it.
3. Add the closing `[RESURFACING <date>] Cancelled ...` comment on the parent; children get a
   one-liner pointing at the parent.

### C-verify (mandatory)

Re-read every touched key: status is Cancelled, Resolution is set, the closing comment landed. Re-run
the C1 query; it should come back empty. Report the reconciled list: cancelled, kept, and anything
that failed mid-batch (name the exact key; a partial batch is reported as partial, never rounded up
to done). Cancelled is not deleted: the trail (comments, labels, history) survives; that is the point
of using a status.
