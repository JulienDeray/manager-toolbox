# Jira reconciliation: detailed procedure

> Capability 1 of the `standup-processor` skill. Matches what was said in the daily stand-up
> against actual Jira state, and builds the writes needed to close the gap (which of them fire is
> set by the Write classes table in SKILL.md). Examples below use
> two invented project keys, `PROJ` and `OPS`; substitute your own from
> `jira_project_keys` in TEAM_CONTEXT.md.

## Contents

- R1: Parse the transcript
- R2: Fetch Jira context
- R3: Match updates to tickets
- R4: New tickets
- R5: Context comments
- R5b: Description & label hygiene
- R5c: Direction linkage
- R6: Build the run report
- R7: Execute and verify

## R1: Parse the transcript

Per team member (names and Slack IDs from the `team_roster` in TEAM_CONTEXT.md, read in Step 0):

- Extract explicit ticket references (`PROJ-101`, `OPS-27`) and implicit ones ("the notification
  work", "the deployment-tooling migration").
- Note blockers ("blocked by", "waiting on"), completions ("done", "shipped", "merged"), and
  status changes.
- List lines from unrecognised names on the Not written list rather than guessing a match.

## R2: Fetch Jira context

Stand-up cares about work actually being pushed **this week**, not the full active/backlog set,
so query narrower than a generic "all active tickets" JQL. Query **per project using named
statuses**, not `statusCategory`, when your workflow distinguishes a committed-but-not-started
column (e.g. "Next") from earlier backlog stages:

- `project = PROJ AND status in ("Next", "In Progress")`
- `project = OPS AND status = "In Progress"` (adjust to each project's actual workflow; if your
  ontology page documents a status mapping, use it instead of guessing)
- Recently transitioned, both projects:
  `project in (PROJ, OPS) AND status changed DURING (-24h, now())`

**Run these through the Jira REST API, not the MCP search tool.** MCP JQL search results
silently truncate two ways: bulk result sets spill to an overflow file (and with a grouping
`ORDER BY`, whole status buckets can be dropped), and some implementations cap at a handful of
rows while ignoring your `fields` list. A stand-up query easily returns 100+ issues, so the
truncated path silently reconciles against a partial board and R3 reports real updates as
"unmatched" (helper script recommended; see the README caveats).

Keep the requested fields narrow (`key,summary,status,assignee,labels,updated,parent`):
`description` bodies are what blows the context budget on a 100-issue result. If the REST path
is unavailable, fall back to the MCP tool but **split per named status** (one query per status
per project) rather than one bulk query, and check each slice's count looks plausible before
matching against it. If the REST path was unavailable and the fallback ran, new-ticket creation
drops to Not written for this run: a partial board makes every unmatched update look new.

For each blocker mentioned, check the ticket's remote issue links for an existing linked page or
related ticket before proposing a new one.

## R3: Match updates to tickets

- Associate each parsed update with a specific ticket from R2's results.
- List ambiguous matches (more than one plausible ticket) on the Not written list, with the
  candidates; never guess.
- List anything that couldn't be matched at all as "unmatched" rather than silently dropping it.

## R4: New tickets

When someone describes real work with **no matching ticket** in R2 or R3, build a new ticket. It
fires only under the conditions in the SKILL.md Write classes table; otherwise it goes on the Not
written list.

Two conventions worth copying into your own TEAM_CONTEXT.md so they travel with the skill:

> When creating new tickets, choose the project by the work area or the assignee's sub-area, or
> by an explicit ticket key; ask if ambiguous rather than defaulting.
>
> Ticket descriptions should be kept up to date: if a ticket lacks a description and context is
> available, propose adding one.

Write the description in terse `lite` style (durable Jira prose: keep articles and full
sentences, drop filler and hedging).

**Direction link.** When the work's link to the current team direction is clear, name the
candidate direction label (e.g. one `axis::` label) in the proposal, at most one, from the
direction page's active subset (read in Step 0). When no link is apparent, say so in the
proposal: a flag for the run report, not a block on creating the ticket. Skip this entirely if
you don't maintain a direction page.

## R5: Context comments

When a ticket's current Jira state doesn't reflect something said in stand-up (extra context, a
decision, a nuance on a blocker, a reason a transition hasn't happened yet), write a comment
with the `standup_comment_prefix` (e.g. `[STANDUP DD-MM-YYYY]`), rather than forcing a bare
status transition that would lose the nuance.

Write the comment in terse `full` style. This is a deliberate exception to the usual "Jira prose
stays `lite`" rule: a stand-up comment is a daily working note, not durable documentation, so
the tighter register is appropriate here specifically, and only here; new-ticket descriptions
(R4) stay `lite`.

## R5b: Description & label hygiene

Beyond comments, the stand-up sometimes establishes a **durable fact** a ticket should carry
itself: a decision, constraint, or scope change missing from the **description** (or a missing
description entirely), or **labels** that don't match your ontology. The comment-vs-description
line: a *daily working note* stays an R5 comment; a *fact that should outlive the week* belongs
in the description.

Prefer composition over inlining: if the ticket-hygiene skill is installed, collect the
candidates (ticket key + the transcript line that prompted each) and hand them to
`ticket-hygiene auto` after this skill's own writes; the run report lists what it applied and
what it returned. The R6 table lists hygiene candidates as "hand-off" rows, not as writes of this
skill.

**Fallback:** without such a skill, do not improvise ad-hoc edits mid-run; list the candidates
(key + reason) under the follow-up suggestions (Capability 3) for a deliberate later pass.

## R5c: Direction linkage

Discussed work should link to the current team direction. For each **discussed** ticket, resolve
its epic (the `parent` field is already in R2's field list); if the epic (or the ticket itself,
when it is an epic or is epic-less) carries **no direction label from the active subset**, list
it as a hygiene hand-off candidate: "committed epic missing its direction label". The check is
epic-level only (the direction obligation lives where work is committed), and, like R5b, applies
to **discussed tickets only, never a board-wide audit**. Skip if no direction page is
configured.

## R6: Build the run report

List every write as one table:

| Ticket | Summary | Epic | Type | Preview | Outcome |
|--------|---------|------|------|---------|---------|
| PROJ-201 | Ticket summary | Epic name | New ticket | First ~50 chars of description... | Written |
| PROJ-142 | Ticket summary | Epic name | Comment | First ~50 chars of comment... | Written |
| PROJ-118 | Ticket summary | Epic name | Hygiene hand-off | Candidate: description misses the region-by-region decision | Not written |
| PROJ-090 | Ticket summary | Epic name | Hygiene hand-off | Candidate: epic missing its direction label (R5c) | Written |

The Outcome column reads Written, Would write (in `dry-run`), or Not written. List unmatched
items and ambiguous matches separately, below the table.

## R7: Execute and verify

In sequence:

1. Apply writes (comments, new-ticket creation) in sequence.
2. **Verify-read** every affected or created issue afterward: parallel write responses can come
   back shuffled, so don't trust the write response alone.
3. Track and report any failures at the end for manual resolution; continue processing the rest.
