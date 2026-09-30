# Status scan + nudge: detailed procedure

> Capability 2 of the `migration-tracker` skill: the gated per-migration rollup (S1 to S5) that
> proposes register-cell updates, channel-intake applies, and nudge drafts, then executes what you
> approve. The unattended daily check reuses S1 to S3 read-only (see
> [nudge-check.md](nudge-check.md)).

## Capability 2: status scan + nudge

A scan answers, per migration: how far along is each expected team, is it on track for its deadline,
and who needs a nudge this fortnight? It never moves tickets, assigns work, or chases teams
automatically: it produces a previewed plan (register-cell diff plus drafted nudges) and waits.

### S1: Select & read

A slug argument scans one migration; no argument lists the active register rows (Status != Done) and
asks, or scans all. From the register row capture Last comms, Next nudge due, Progress, Status, Stage,
deadlines, and the epic key (keep the page version for the cell update). From the epic: the
expected-teams list and the cadence. From the detail page: per-team overrides (a "Not impacted" marker
with date + permalink, a claims-complete annotation) and the comms-log permalinks (the intake dedup
set).

### S2: Collect associated work (read-only, must not truncate)

Pull every ticket associated with the migration across all projects via both mechanisms:

- label: `labels = "migration::<slug>"` (cross-project),
- epic-parent: the children of the tracking epic.

Run both through the paginated REST search (see Requirements in [SKILL.md](../SKILL.md)). Dedup by issue key (a ticket may carry
both). Keep fields narrow: key, project, statusCategory, updated (description bodies blow the context
budget). Compare the issue count against the expected-teams denominator as a sanity check.

**If the paginated search is unavailable**, fall back to the MCP but never to a single bulk query:
split the JQL into slices small enough that none can truncate (project x issuetype x statusCategory),
add one `project not in (<expected projects>)` catch-all to prove nothing was missed, reconcile the
slice counts, and say in the run log that the rollup was hand-reconciled.

**Ownership split before grouping.** The rollup groups by project key, so any team whose migration
work lives on another team's board must be re-keyed onto a synthetic project matching its
expected-teams entry first, or it reads as not started while the host team is inflated. E.g. a team
whose tickets sit in a host project under its own label: key those tickets by that label onto the
team's expected-teams entry before grouping.

### S2.5: Channel intake (gated here)

Run the nudge check's intake procedure (C3.5 in [nudge-check.md](nudge-check.md)): collect nudge-thread replies and recent
migration mentions, permalink-dedup against the comms log, classify with the confidence split. But
under this capability's normal gate: nothing is applied here; the proposed applies join the plan for
approval (clear-cut ones pre-marked, hedged ones as capture-only rows with the open question stated).
Intaken overrides (`not_impacted`, `claims_complete`) feed the rollup and the nudge wording.

### S3: Compute

From the collected facts, compute deterministically (helper script recommended; see the README caveats):

- per-team state: `not_started` / `in_progress` / `done` / `not_impacted`, plus last activity;
- the progress fraction: numerator = done + not-impacted; the denominator never shrinks;
- a **suggested** RAG status with a reason (e.g. At risk when days-to-hard-deadline < remaining teams
  x cadence, or when past the soft deadline; Stalled when nothing moved over a full cadence);
- nudge-due (`today >= last comms + cadence`) and the nudge targets: lagging teams only, never done
  or not-impacted ones;
- proposed register cells.

**Always honour a nudge hold.** The register's Next nudge due cell is how you postpone a round
without touching the cadence (owner on holiday, release week): pass it into the computation as a
hold. A hold can only push a nudge out, never pull it in, and the next real send clears it. When a
hold is in effect, say so in the plan rather than silently reporting "not due"; an unread hold means
the run drafts anyway and the postponement is fiction.

### S4: Present the plan

- **Per-team rollup table**: team, state, done/total issues, last activity, association mechanism.
  Call out unexpected projects (a labelled ticket from a non-expected team: someone associated work
  you didn't anticipate, or a typo'd label; decide whether to add the team, don't silently fold it
  into the percentage) and roster gaps.
- **Intake applies** from S2.5, each justified by the quote + permalink that prompted it; strike or
  amend before approval.
- **Register-cell diff**, before and after. Status shows the suggested RAG with an explicit override
  line. Last comms / Next nudge due **never change in the scan**: the skill does not send, so the nudge
  check's send-detection (C3 in [nudge-check.md](nudge-check.md)) moves them once it sees your send.
- **Drafted nudges, both surfaces, you pick which to draft**: one short message per lagging team to
  that team's channel (from the roster), owner tagged, stating the team's current state, the hard
  deadline, and the self-serve/runbook link, carrot-first (help offered, not just a deadline), shaped
  by the intake annotations (acknowledge a logged blocker; redirect a claims-complete team to closing
  and labelling its ticket instead of asking it to start); plus one consolidated summary to
  `team_channel`. Drafts are created only for the surfaces you pick (one, both, or none); the manager
  sends them from Slack.
- **Failure-mode check**: one line against the playbook checklist (stalled at 80%, owner present,
  deadline teeth, visibility).

If nudges are not due, say so: propose the Progress/Status refresh only and offer a manual nudge as
explicit opt-in.

### S5: Execute + verify

1. Create drafts for the approved nudges via the draft tool, one per target channel (one attached
   draft per channel, as in C4); the manager sends them from Slack. If the draft tool is
   unavailable, the text in chat is the deliverable. Never call the send tool. Skip the Last comms /
   Next nudge due bump entirely: the cadence clock only advances on a real comms, which the nudge
   check's send-detection (C3) reconciles once you have sent (run `/migration-tracker check` after
   sending if the daily check is not scheduled).
2. Apply approved intake writes to the detail page (one page update, targeted at the right tables).
3. Update the register row in place: re-supply all cells so the row is rewritten consistently,
   Last comms / Next nudge due keeping their pre-scan values, Status macro
   coloured by the (possibly overridden) RAG.
4. Verify-read: the row shows the new cells, other rows and the header intact, row count unchanged,
   version +1; the detail-page writes landed (a lost comms-log row breaks the next intake's dedup);
   each approved nudge draft exists in its channel. Report per migration: progress, status (and
   whether you overrode it), nudges drafted or held (and where each draft sits), flagged failure
   modes.

Notes: no associated tickets yet is a legitimate early-Derisk state (every team not started; nudge
them, if due, to create their tickets). Absence of an epic-parent link on a team-managed team is
expected; rely on the label and never report it as missing. A `0/?` denominator means no percentage
and no confident RAG; fix the expected-teams list first.
