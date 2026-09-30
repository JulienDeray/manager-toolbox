---
name: stepback-processor
description: |
  Wraps up the team's quarterly step-back session, the planning slot where the team reviews its
  long-term direction board instead of running the weekly grooming. Extracts the alignment verdict,
  decisions and actions from the session notes, appends the history entry to the Quarterly
  Step-Back Confluence page, bumps the next step-back date, then invokes planning-processor on the
  same notes. The page write waits for the operator's approval.
  Use after a quarterly step-back session, when processing step-back notes, or when the user
  mentions wrapping up or capturing the step-back, the direction-board review session, or
  bumping the next step-back date.
---

# Quarterly Step-Back Processor

The **wrap-up** of the quarterly step-back ritual. Once a quarter, the weekly planning slot
becomes a session where the team reviews its long-term direction board together and checks
day-to-day priorities against the long-term direction. This skill records the outcome. It is the
once-a-quarter sibling of `planning-processor`: the operator runs **this skill instead of** the
processor on a step-back week, and this skill **invokes planning-processor itself** at the
end, so one invocation covers the whole session.

Two effects, in order:

1. **Capture**: append the session's row (Date · Verdict · Verdict notes · Decisions/themes ·
   Actions taken) to the Quarterly Step-Back page's History table, and bump **`Next step-back`**
   to the first planning weekday (the weekday of `planning_slot`) on or after session date + 3
   months. Besides confirmed glossary rows in `TEAM_CONTEXT.md`, this single page update is the
   skill's **only write**.
2. **Hand off**: invoke the `planning-processor` skill on the same session notes for the regular
   weekly machinery (focus labels, wrap-up draft, hygiene), under its own gates.

**The bump is what stops the nag:** `planning-agenda` renders step-back mode every week while
`Next step-back` sits in the past. If this skill is forgotten, that nag is the recovery signal.
**Never bump the date without also writing the entry**: a bumped date with no entry silently
kills the nag AND loses the capture.

No Slack writes, no Jira writes, no board-tool API calls: the board is reviewed by humans in the
session; this skill only records the outcome.

## Requirements

- **Confluence MCP tools**: get page, update page.
- **Plan mode** (or an equivalent explicit approval step) for the write gate.
- **The `planning-processor` skill** for the hand-off, a soft dependency: if it's unavailable,
  still land the capture, then report that the weekly planning pass is pending; **never inline**
  its writes (focus labels, wrap-up drafts) here.

## The step-back page contract

The Quarterly Step-Back page (`stepback_page` in TEAM_CONTEXT.md) carries two tables:

- **Session config** (first table, two columns): a `Next step-back` row with the date, a row
  linking the long-term direction board, and the session instructions (read verbatim by
  `planning-agenda` when composing the step-back announcement).
- **History** (second table, under a `History` heading, five columns): Date · Verdict · Verdict
  notes · Decisions / themes · Actions taken. **Newest row first**: `planning-agenda`'s "what
  was discussed last time" is the first data row, so newest-first ordering is an invariant. A
  fresh page carries a placeholder row ("No step-back sessions yet") that the first real entry
  removes.

**Verdict vocabulary**, exactly one of:

- `aligned`: day-to-day work points at the board's direction.
- `drifting-reactive`: the work is real but reactive; the board isn't steering it.
- `board-is-wishful`: the board no longer describes reality; it needs rework, not the work.

Render the verdict as a coloured status cell if your page uses status macros (green / yellow /
red respectively).

## Step 0: load team context

Read **`TEAM_CONTEXT.md`** at the repo root and take the `stepback_page` id (never hardcode it).
If the field is absent, **stop**: this skill is meaningless without the page. Point the operator
at the bootstrap: create the page per the contract above and register its id in TEAM_CONTEXT.md.
Any other config-load failure is also a hard stop. Also take `date_format`, `planning_slot` (its
weekday is the day the bump lands on), and the transcript-garble glossary (for the notes-hygiene
decode in step 1).

## Flow: plan, approve, execute, verify

1. **Gather (plan mode).** Read the step-back page: body, version number, current
   `Next step-back`, and the newest History row. From the notes, extract:
   - the **actual session date** (explicit in the notes, else today);
   - the **verdict**, exactly one of the three vocabulary values. **If the notes don't support a
     clear call, ask the operator; never guess** the alignment verdict;
   - the 1-2 sentence **verdict notes**;
   - the **decisions/themes**;
   - the **actions taken** (explicitly "none" if none).

   **Notes hygiene:** when the session notes come from auto-transcription, follow the transcript
   hygiene conventions in `TEAM_CONTEXT.md` and decode before extracting (mangled names and
   product terms skew the verdict evidence). New glossary rows are proposed in the plan and
   appended to the glossary table only once confirmed.

   **Idempotency:** if the newest History row already carries this session date, stop and ask;
   the session was likely already processed.
2. **Compute.** Validate the entry (verdict in vocabulary, no empty fields) and compute the
   bump: session date + 3 months (day-clamped month addition), rolled forward to the next
   planning weekday (a date already on that weekday stays). This is deterministic, so every
   quarter is computed identically (helper script recommended; see the README caveats). A
   validation failure means
   fix the extraction (or ask), never force the write.
3. **Approve.** Show the exact History row, the bump as `old date to new date` with its
   derivation (e.g. "<session date> + 3 months = <date>, rolled forward to the planning
   weekday"), any proposed glossary rows, and that `planning-processor` will be invoked next on
   the same notes under its own gates. Approval authorises exactly **one Confluence page
   update**, the confirmed glossary rows, and that invocation, nothing else.
4. **Execute + verify** (mechanics below).
5. **Hand off.** Invoke the `planning-processor` skill with the same session notes (invoke,
   don't inline: it runs its own Step 0, plan mode, and approval gates). If unavailable: report
   that the step-back capture landed and the planning pass is pending.
6. **Report.** The entry, the new `Next step-back` date, the page link, and what
   planning-processor did (or that it's pending).

## Write mechanics

### W1: read the page

Fetch the page body (HTML/storage format) and the version number. Reuse the plan-mode read if
still fresh (no other writer touched the page since); re-read if in doubt.

### W2: splice the History row

Insert the five-cell row **at the top of the History table** (directly after its header row):
newest-first is the invariant `planning-agenda` relies on. On the first real entry, also remove
the "No step-back sessions yet" placeholder row (a no-op afterwards).

**Target the right table.** The page has **two** tables (Session config first, then History), so
a naive "first table" splice lands the session entry in the Session-config table. Anchor the
splice to the `History` heading. A row-width sanity check catches this class of mistake: a
5-cell row against Session config's 2-column header is a stop, not a warning. Do an anchored,
byte-safe table splice (helper script recommended; see the README caveats). If editing by hand,
change only the inserted bytes and diff before writing.

### W3: bump `Next step-back`

In the same new body, update the Session-config `Next step-back` cell **in place** to the
computed date. The Session-config table is the first one and the row is found by its label, so
this edit can't wander into History. Never perform this edit without the W2 row in the same
update.

### W4: update the page (one write)

One page update with the new body, `version = previous + 1`, and a message like
`Step-back <session date>: entry + next bumped to <new date>`.

**Storage-format caveat** (same as every table edit in this skill family): round-trip the body
in the format you fetched it and splice only the targeted bytes, so every other byte of the page
is left alone.

### W-verify (mandatory)

Re-read the page and confirm:

1. The new row is the **first data row** of the History table, all five cells exactly as
   approved (verdict status colour correct, if used).
2. Prior History rows and the rest of the page are intact; the placeholder row (if it existed)
   is gone.
3. The `Next step-back` cell shows the new date.
4. The version incremented by exactly 1.

Report the reconciled result plainly: entry, old and new date, page link. Only claim success
once the verify-read matches the approved plan; then proceed to the planning-processor hand-off.

## Usage

After the step-back session, with the notes pasted or pointed at:

```
/stepback-processor <notes or a pointer to them>
```

The skill loads config, extracts and validates the entry, presents the row + bump for approval,
writes the page, then invokes planning-processor on the same notes.

## Notes & edge cases

- **Version conflict** (someone edited the page between read and write): re-read, re-splice on
  the fresh body, retry the update once. Never hand-merge HTML.
- **Re-run protection is upstream:** the plan-mode idempotency check (newest row already carries
  this session date) is what prevents duplicate rows; if you somehow write twice for the same
  session, the verify-read will show the duplicate: undo by hand, don't script a deletion.
- **A hand-edited page is fine.** Someone may have reworded the instructions or edited a past
  row; the scoped splice leaves those bytes alone. Only the two targeted edits above are yours.
- **A hand-edited verdict in an old row is displayed, never validated or rewritten**: the
  vocabulary check applies to the row you are writing, not to history.
- **No other writes.** No Slack, no Jira, no board tool from this skill (the confirmed glossary
  rows aside); the weekly machinery belongs to the invoked `planning-processor`.
