---
name: ticket-hygiene
description: |
  Proposes description updates and label fixes on existing Jira tickets a conversation touched,
  when a discussion established a durable fact the ticket doesn't carry or its labels don't match
  your ontology. Preview-then-approve, never a board-wide audit. One caller-only exception: an
  `auto` mode, for a calling skill that declares append-only hygiene as a firing write class
  (standup-processor), applies only append-only edits and returns the rest.
  Use when tidying ticket descriptions or labels, retagging a ticket ("fix the labels on
  PROJ-101", "clean up these tickets"), or folding meeting or thread context back into tickets.
---

# Ticket Hygiene

Keeps discussed tickets honest: when a conversation (a planning session, a stand-up, a Slack thread,
or a direct ask) surfaces things a ticket itself doesn't carry (a decision, a constraint, a scope
change missing from the description, or labels that don't match the ontology), this skill turns them
into proposed ticket edits, preview-then-approve, never silent. Its only write surface is the
description and labels of existing tickets in your team's projects. It operationalises a simple team
convention: ticket descriptions should be kept up to date, and if a ticket lacks a description and
context is available, propose adding one.

**One procedure, many callers.** If you have planning or standup processing skills, have them invoke
this skill with their discussed tickets plus the note lines that justify each candidate, rather than
inlining their own copy of the procedure (which drifts). This skill runs its own approval gate
regardless of who invoked it, with one exception: a calling skill whose Write classes table
declares these hygiene edits as firing (standup-processor's hand-off) may pass `auto`, which
applies only the safe edit classes in [Auto mode](#auto-mode-no-gate) and returns everything else
to the caller. Standalone runs and gated callers keep the gate, always.

## Requirements

- **Jira MCP tools**: get issue, edit issue.
- **Plan mode** (or an equivalent approval gate); not used in `auto` mode.
- Any JQL search used to *find* tickets from a description should go through a paginated Jira REST
  search rather than an MCP search tool, which can silently truncate result sets: a silent row cap
  would drop tickets from the batch without saying so, and the whole promise here is a batch the
  operator can see in full (helper script recommended; see the README caveats).

If a required tool is missing, say so and stop rather than working around it.

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root:

- `jira_project_keys`: the team's project keys. **Team projects only**: never edit another team's ticket
  (e.g. a migration participant's ticket mentioned in passing).
- `team_ontology_page`: the label axes and their heuristics (work-type `wt::`, area `area::`, service
  `svc::`, strategic `axis::`, plus any carve-outs and retired labels).
- `team_roster`: name to Jira account ID, for assignee-informed area proposals.

When invoked by a sibling skill that already loaded this configuration in the same session, reuse it.
If context loading fails, stop and say which field is missing.

## Flow: plan, approve, execute, verify

H1 to H3 are read-only and build the candidate edits; H4 presents them as one plan; H5 executes
only after explicit approval, then verify-reads. The only exception is [Auto mode](#auto-mode-no-gate),
which a declaring caller passes explicitly.

## Scope

- Only tickets the input actually concerns: explicit keys, or items confidently matched from the
  notes. This is not a board-wide audit; it fixes what the conversation touched. An ambiguous match
  ("the config ticket", two candidates): ask, don't guess.

## H1: Collect the candidate set (read-only)

From the input (keys + notes, or a caller's hand-off), list every ticket with the source lines that
concern it, then read them all: summary, labels, status, assignee, and the description **as flattened
text**.

Read descriptions the safe way: two read traps (a rendered field that was never requested, an
ADF object judged without flattening) both make a full description look empty, and H2 would then
overwrite it.

**Full procedure:** [references/description-reads.md](references/description-reads.md).

## H2: Description check

Compare what the input established against the current description:

- **Never conclude "empty" from a falsy read.** This branch is the destructive one, so "empty" needs
  a positive reading: a flattened text that is genuinely blank. A null, an empty object, a length of
  3, or a field that was never requested is a bad read, not an empty ticket; re-read before going
  further. Treat an "empty" description on a ticket that has clearly been worked (assigned, in
  flight, long changelog, a summary implying prior write-up) as a bad read until a clean re-read says
  otherwise; the issue changelog settles it, since a prior description shows up there. A caller's
  hand-off row saying "no description" is a claim, not a reading: H1's read decides. When still
  unsure, propose the **additive** edit: the worst case is a slightly redundant paragraph, not lost
  text.
- **Empty or placeholder description**: draft one from the input plus the summary.
- **Material point missing** (a decision made, a discovered constraint, a scope change, a
  dependency): draft an additive update that preserves the existing text and integrates the new point
  where it belongs; don't bolt a raw transcript quote on the end.
- **Not material** (status chatter, who picks it up next): skip; that is what ritual comments and the
  board are for. When a caller also posts a session comment on the item, the comment carries the
  session's intent; the description carries the durable facts.

Write descriptions tight but in full sentences (they are a durable record). The plan shows a
before-and-after diff with the source line(s) that justify it.

## H3: Label check (against the ontology from Step 0)

- **Exactly one work-type label.** Missing: propose one via the ontology's priority-ordered
  heuristics. Two present: propose which to drop. Genuinely torn: flag for grooming instead of
  guessing. Honour the ontology's carve-outs (e.g. an area label that replaces the work-type for
  certain work, or an epic-only label never proposed on an ordinary task). A straggler retired label:
  propose the documented replacement.
- **Area label**: propose only where the sub-team is clear from the input or assignee; optional.
- **Service label**: propose only when the service is explicit; never infer from vibes.
- **Board-mechanic labels are bugs to flag, not untidiness.** If your boards derive swimlanes or
  filters from labels or fixVersions, a wrong value silently hides or misplaces the item (an ops-board
  label on the wrong issue type drops it from the board; a released fixVersion hides an epic behind a
  board sub-filter). Flag these loudly. But respect the design's defaults: if your lane model treats
  "no lane label" as a valid default lane, an unlabeled epic is not a missing-label finding.
- **Commitment-line labels** (e.g. a strategic `axis::` label your team requires on committed epics):
  propose one only when the input makes the choice clear, otherwise flag for grooming. On backlog
  items such labels stay optional; don't propose them.
- **Read-modify-write, no-clobber**: fetch the current labels and add or remove only the label under
  discussion; never touch other namespaces (priority, migration, topic, or the other axis labels not
  at issue).

## H4: Present the plan

One row per proposed edit: `Key · Field (description / labels) · Before -> After · Why (source
line)`. Description diffs shown in full; label changes as full before-and-after arrays. The operator can
strike any row: hygiene edits are suggestions drawn from the conversation, not corrections the skill
is sure of.

## H5: Execute + verify (after approval)

Sequential edit calls (parallel responses can be shuffled and misattributed), then re-read each edited
issue and confirm description and labels landed exactly as approved before reporting success.

**The verify-read carries the H1 traps in their nastiest form**: a verify that reads the field the
wrong way confirms a state it never saw (a null from an unrequested rendered field, or a substring
check against an ADF dict, both "pass" no matter what is on the ticket). Re-read with the same
flattening read as H1 and compare the flattened text against the approved after-text; a verify that
reports "empty" or that can't find the text it just wrote is a read failure until proven otherwise.
Then hand the outcome (keys edited, keys flagged-for-grooming) back to the caller if one invoked this
skill.

## Auto mode (no gate)

Only when the caller passes `auto`: never on a standalone run, and never on the skill's own
initiative. `auto` exists for one situation: a calling skill whose own write-class table
declares these edits as firing; standup-processor is the only one. A gated caller
(planning-processor) never passes it; its hand-off goes through H4.

H1 to H3 run exactly as above. H4 is skipped, and each candidate edit is sorted into one of two
lists:

**Applied (append-only safe classes only):**

- **Label additions**, read-modify-write, no-clobber, and only where H3 is confident: an
  explicit service label, a clear area label, one work-type label when none is present, or a
  commitment-line label on a committed epic when the input makes the choice clear. Additions
  only; a removal or a swap is never safe here.
- **Additive description edits**: the existing description is preserved byte-for-byte and one
  new dated paragraph is appended at the end (`Update <date> (<caller>): ...`), never
  integrated into, reworded, or reordered. Append it as one new paragraph node at the end of
  the fetched rich-text (ADF) content and write that document back; never resend the
  description as retyped text, because a retype is where drift happens.
- **Empty-description drafts**, only when the flattened H1 read positively reports the
  description empty AND the changelog check shows no earlier description ever existed. Either
  signal missing: not written.

**Returned to the caller as not written:** label removals or swaps, a second work-type label or
a work-type conflict, board-mechanic bugs (H3's flag-loudly cases), anything flagged for
grooming, a low-confidence commitment-line label, and any description change that would alter
existing text or where the input conflicts with it. Each item returns with its key, the
proposed edit, and the source line, so the caller can list it for the operator. Anything
judgment-heavy or ambiguous belongs on this list, not in a best-effort write.

H5 runs unchanged on the applied edits. For every edited description, also hand the caller the
full before-text from the H1 read, so the operator can roll back from the caller's run report.
The verify passes only when the after-text starts with the before-text unchanged.

**`auto dry-run`** (passed through by the caller's own dry-run mode): sort exactly as above and
return both lists, the applied set marked "would apply", but write nothing and skip H5.

## Usage

```
/ticket-hygiene PROJ-101 has no work-type label; PROJ-142: we agreed rollout is region-by-region, EU first, description doesn't say it
/ticket-hygiene [paste meeting notes]     # the skill extracts the tickets + candidate edits
/ticket-hygiene PROJ-101                  # the skill asks what prompted the look
```

Or invoked by your planning or standup ritual skills at the end of their runs, with the
discussed-tickets list and note lines pre-filled. A caller that declares hygiene as a firing
class appends `auto` (Auto mode above); gated callers don't, so their hand-off keeps the gate.

## Notes & edge cases

- **Recovering a clobbered description**: the prior text survives in the issue changelog, in Jira
  wiki markup. **Full procedure:** [references/description-reads.md](references/description-reads.md).
- **Conflict with existing content**: if the input contradicts what the description already says,
  present both versions and ask; never overwrite silently.
- **A discussed ticket that doesn't exist yet** is not hygiene; that is an intake / creation path
  (see the `intake` skill). Say so.
- **No transitions.** Hygiene never changes a ticket's status; a ticket that looks done or dead is
  flagged for the operator (a cancellation goes through `backlog-resurfacing cancel`).
- **Zero proposals is a fine outcome.** Most conversations won't move descriptions. Don't manufacture
  edits to have something to show.
