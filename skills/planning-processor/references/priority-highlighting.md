# Priority highlighting: detailed procedure

> Capability 1 of the `planning-processor` skill: the weekly **clear-and-reapply** of the focus
> label (`prio::now` in the examples) plus a native **Priority** bump on the top 1-3 focus
> items, and a planning comment on each focus item. Examples use invented project keys `PROJ`
> (the primary project, verified priority scheme) and `OPS` (a secondary project on the same
> epic board, unverified priority scheme).

## Contents

- Inputs
- P1: Snapshot current state (read-only)
- P2: Compute the diff
- P3: Build label arrays (no-clobber rule)
- P4: Present the plan
- P4b: Planning comment per focus item
- P5: Execute
- P6: Verify-read (mandatory)
- Notes & edge cases

All writes happen plan-mode, approve, execute, verify. Nothing below writes to Jira before the
user approves the plan.

> **Read the actual priority scheme before writing.** Jira projects vary: one team's scheme is
> MoSCoW (**Must / Should / Could / Won't**: bump = Must, reset = Should), another's is
> Highest/High/Medium. Never assume the Jira-default names; read an issue's current priority
> name first and use the scheme you find. **Projects with an unverified scheme never get a
> Priority write**: label and comment only.

## Inputs

This week's **agreed focus**, as a *ranked* list, from the planning notes (or asked from the
user if absent):

- **Focus epics**: epic-type issues on the epic planning board. Usually the bulk of the focus.
- **Key tickets**: a few delivery items (Story/Task on the delivery board) worth surfacing in
  their own right.
- **Ranking**: order matters only enough to identify the **top 1-3**, which also get the
  Priority bump (verified-scheme projects only).

Selection rules live on your team ontology page (read in Step 0); that page is canonical, don't
restate it here. The focus pass doubles as a cheap alignment checkpoint: check each focus item's
labels against the ontology, especially whatever axes encode the team's scope or strategy. A
focus item whose labels place it outside what the team says it is here to do deserves a flag in
the plan: either the labels are stale, or the team is about to spend its week on off-mission
work, which should be a deliberate call rather than an accident. Never auto-fix a label here;
hand it to ticket-hygiene if the session discussed it.

If the notes name items by description rather than key, resolve them to keys with a JQL search
(e.g. match against the ontology page's active-epics list) and confirm the matches in the plan
before writing. Cap the focus list to a small set: the routine exists to **reduce** WIP, so a
focus list longer than ~8 items is a smell; flag it rather than silently applying.

## P1: Snapshot current state (read-only)

1. **Last week's holders.** Find everything currently carrying the label:

   ```
   project in (PROJ, OPS) AND labels = "prio::now" ORDER BY priority DESC, key
   ```

   For each result capture: key, summary, **full `labels` array**, current **priority**, status.
   These are the items that may need the label removed and/or Priority reset.

   **Run this via the Jira REST API, not the MCP search tool**: a focus set of 6-8 items already
   exceeds some MCP searches' silent row cap, and a missed holder is a label this routine fails
   to clear, so last week's focus would bleed into this week's board colouring with nothing to
   show it happened. The P6 verification re-runs this query, so it inherits the same
   requirement (helper script recommended; see the README caveats).

2. **This week's targets.** For each proposed focus item, read the issue to capture its **full
   `labels` array** and current **priority**. The full label array is mandatory: see the
   no-clobber rule in P3.

## P2: Compute the diff

Treat last week's holders (set **H**) and this week's focus (set **F**) as sets of issue keys:

| Bucket | Members | Label action | Priority action (verified projects only) |
|--------|---------|--------------|-------------------------------------------|
| **Clear** | H minus F | remove the focus label | if currently bumped, reset (see below) |
| **Apply** | F minus H | add the focus label | top 1-3 get the bump value |
| **Keep** | H intersect F | none (already labelled) | top 1-3 get the bump; others reset if currently bumped |

**Priority bump cap = top 1-3 only.** If the user wants more than 3 bumped, do not: apply the
label to all focus items but bump only the top 3, and note the cap in the plan.
**Unverified-scheme keys are exempt from every Priority action in this table**: such an epic in
the top 1-3 keeps its Priority untouched (note it in the plan).

**Priority reset value.** When an item leaves the top 1-3 (dropped from focus, or demoted),
reset its Priority. The skill keeps no cross-week state store, so it cannot always know the
*original* value. Rule:

- If the P1 snapshot still shows the pre-bump value (rare), restore it.
- Otherwise reset to the project's default (e.g. Should in a MoSCoW scheme) and **flag the reset
  in the plan** so the user can correct any item that should keep a non-default Priority.
- Never reset the Priority of an item that is *not* part of this week's clear/keep buckets;
  leave unrelated issues alone.

## P3: Build label arrays (no-clobber rule)

A Jira edit replaces the whole `labels` array, so **never write a bare `["prio::now"]`**.
Tickets carry other meaningful labels (work-type, service, area, topic labels, per your
ontology). For each write:

```
new_labels = existing_labels with "prio::now" added (Apply) or removed (Clear)
```

All other labels are preserved verbatim. In the plan, show **before and after** label arrays for
every item so the user can see nothing else moved.

## P4: Present the plan

One table covering every write, e.g.:

| Key | Summary | Bucket | Label: before, after | Priority: before, after |
|-----|---------|--------|-----------------------|--------------------------|
| PROJ-243 | Secrets self-management | Apply (top-1) | `area::team, axis::reliability` gains `prio::now` | Should to **Must** |
| OPS-112 | On-call toil reduction | Apply (top-2) | `area::team` gains `prio::now` | (unverified scheme: no Priority write) |
| PROJ-207 | Annual pentest | Keep | (unchanged) | Must to Should *(dropped from top-3)* |
| PROJ-187 | HTTP client upgrade | Clear | `wt::long` loses `prio::now` | Must to Should |

Include a one-line summary: *"Focus this week: N items (was M last week). +X labelled, -Y
cleared, top 3 bumped."* Surface any reset-to-default guesses, any focus item flagged by
the alignment checkpoint above, and any unresolved name-to-key matches
here. Do **not** proceed without explicit approval.

## P4b: Planning comment per focus item

For **every** focus item (the Apply + Keep set), post a Jira comment so the board records the
week's intent next to the work:

- **Prefix:** the `planning_comment_prefix` with the planning date (mirrors the stand-up prefix
  convention).
- **Body, two lines:**
  - **Stage:** where the work stands now (from the planning notes / current ticket state).
  - **Expected this week:** the concrete next step(s) committed in the session.
- Keep each tight (a few sentences). Items dropped from focus (the Clear set) do **not** get a
  comment.

**Preview** each comment's text in the P4 plan (next to that item's label/Priority diff) so the
user approves the wording. **Execute** after approval, one call per item. **Verify** the posted
comment ids in P6.

## P5: Execute

After approval, apply each change:

- **Labels:** write the full no-clobber array from P3.
- **Priority (verified-scheme keys only):** write the actual scheme names read in P1. Never
  send a Priority write for an unverified-scheme key.

**Bulk-write caveat:** parallel Jira edit responses can come back shuffled. Prefer applying the
writes **sequentially**; if batched, do not trust the per-call responses, rely on the P6
verify-read instead.

**Per-issue robustness:** if an edit fails (e.g. Priority not editable on an Epic in this
workflow, screen restrictions), do not abort the whole run: record the failure, continue with the
rest, and report failed items at the end so the user can handle them manually.

## P6: Verify-read (mandatory)

1. Re-run the holder query (same REST path as P1; a truncated verify-read would "confirm" a
   state it never saw). The result set must equal **F** (this week's focus). Any extra = a clear
   that didn't land; any missing = an apply that didn't land.
2. Re-read the top 1-3 bumped items and confirm the bump landed; re-read reset items and confirm
   the reset. Unverified-scheme items: confirm the Priority did **not** change.
3. Confirm each focus item got its planning comment (one per Apply + Keep item).
4. Report the reconciled result. Only claim success once the verify-read matches the approved
   plan; list any discrepancies plainly.

## Notes & edge cases

- **Board membership is not touched by this procedure.** The focus label is an issue-level
  label; boards only *display* it (via a card colour or a "This week's focus" quick filter,
  one-time UI setup). The skill just labels issues correctly.
- **A focus-label holder that is now Done** still gets cleared: last week's focus is no longer
  "now" regardless of status.
- **Items outside the configured projects** (a stray key from another team slipped into the
  notes) are out of scope: skip and flag.
- **No "next" label.** Next-up work lives in the board's committed column, never as a second
  label; one focus label is the whole mechanism.
