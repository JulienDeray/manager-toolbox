---
name: intake
description: |
  Your team's single front door for unstructured input (a chat message, a Slack permalink, a meeting
  transcript, a note). It classifies the input against your label ontology, dedups it, and proposes
  a home: a Jira ticket, an Epic, a Backlog idea, or a migration for migration-tracker. Nothing is
  filed until you approve, and the outcome post is a Slack draft.
  Use when someone says "file this", "triage this", or "where should this go", or pastes a message,
  permalink, or transcript to turn into an action.
---

# Intake

The front door for unstructured input. Without one, a chat thread, a meeting takeaway, or a
"we should..." message either gets lost or turns into inconsistent, untagged ticket-making. This skill
takes any raw input, matches it against team context, proposes the right home for it, and, on approval,
files it and announces the outcome.

This is a Jira + Confluence + Slack routine. Everything is preview-then-approve: nothing is written to
Jira, Confluence, or Slack without showing the plan first.

**Scope of writes: narrow on purpose.** Intake writes only to your own destinations: a ticket or epic
in one of your team's projects (an "idea" is just a Backlog Task, see the idea shape below), or, via
the migration-tracker skill, your migration register. After a write it leaves an outcome post for the
team channel as a Slack draft the manager sends. It classifies but never auto-files: every route is surfaced as a
proposal (type + target + payload + confidence) and waits for approval. One input usually maps to one
destination; a small fan-out is allowed only when the input clearly contains several distinct items.
It never edits another team's work, and for the migration branch it hands off to the
`migration-tracker` skill rather than re-implementing onboarding.

## Requirements

- **Jira + Confluence MCP tools** (create issue, edit issue, get issue, get/read transitions,
  read pages).
- **Slack MCP tools** (read a thread or channel to resolve a permalink; the message-draft tool for
  the outcome post; this skill never calls the send tool).
- **Plan mode** (or an equivalent explicit approval gate) for the preview-then-approve flow.
- For the dedup search, a paginated Jira REST search. Many Jira MCP search tools silently truncate
  bulk result sets (a hard row cap with no pagination path, or an overflow file whose ordering can
  drop whole buckets), and a dedup check that cannot see all the candidates misses the duplicate it
  exists to catch (helper script recommended; see the README caveats). If you must fall back to the MCP search, say in the plan that the dedup was best-effort over a
  possibly-truncated result rather than skipping the check.

If a required tool is missing, say so and stop rather than working around it: the writes and the
approval gate depend on these tools.

## Step 0: load team context

Before doing anything, read `TEAM_CONTEXT.md` at the repo root and extract:

- `jira_project_keys`: the team's Jira project keys (the routing targets).
- `backlog_project`: the project whose Backlog holds ideas.
- `routing_rules`: which project takes which kind of work.
- `team_ontology_page` (optional): the label axes and their classification heuristics. Labels here
  are prefixed by axis: `wt::` is the work type (exactly one per issue), `area::` the sub-team,
  `axis::` the strategic direction an item serves, `prio::now` the weekly-focus label, and
  `migration::<slug>` marks cross-team migrations. Substitute your own; the Label ontology section
  of `TEAM_CONTEXT.template.md` has a starter ontology. Without it, propose destination, project and
  status only, skip label proposals, and say so.
- `definition_of_ready`: what an issue needs before it advances past backlog.
- `team_channel`: the team Slack channel for the outcome post.
- `team_roster`: name to Slack ID to Jira account ID mappings, to resolve a raiser or owner and to
  tag the outcome post.
- `jira_site_url`: for browse links.

**If context loading fails, stop and say which required field is missing. Do not proceed with partial
configuration.** Never hardcode channel IDs, project keys, or page IDs in this skill: TEAM_CONTEXT.md
is the single source, so the skill survives renames and re-orgs.

## Flow: plan, approve, execute, verify

1. **Gather + analyse (read-only):** enter plan mode, normalise the input, classify the intent
   against the ontology, run the pre-write dedup search, and build a concrete plan of the single
   proposed write (or small fan-out): type + target + full payload + confidence.
2. **Approve + execute:** present the plan. Only after explicit approval, execute the write. The
   migration branch hands off to the `migration-tracker` skill, which runs its own approval gate.
3. **Verify-read after every write.** Parallel Jira edit responses can be shuffled and misattributed:
   run writes sequentially, then re-read the issue to confirm the writes landed before reporting
   success.
4. **Slack handoff: a human sends.** The outcome announcement is created as a Slack draft in
   `team_channel` via the draft tool; the manager sends it from Slack. If the draft tool is
   unavailable, the text in chat is the deliverable. Never call the send tool.

## Invoked by another skill

Items arrive pre-selected with originator and provenance: skip N3 (selection), keep the passed
provenance, still run classification, dedup and the gate.

## Capability 1: Normalise + classify, then propose

Always runs first. Read-only; it produces the proposal the routing capabilities consume.

### N1: Normalise the input

Detect the shape and resolve it to a clean, self-contained problem statement:

| Shape | How to recognise | How to resolve |
|-------|------------------|----------------|
| Slack permalink | a `slack.com/archives/<channel>/p<ts>` URL | Read the thread to pull the parent plus replies, not just the linked message. A channel link with no thread: read nearby channel context. |
| Pasted chat message | raw text with `@`/`#` mentions, timestamps, "X said:" framing | Use as-is; strip UI cruft (reactions, "edited", join/leave lines). Resolve mentions to canonical names where `team_roster` knows them. |
| Meeting transcript | long multi-speaker text, or a document link | Fetch if it is a link. Summarise to the decision or ask, not the whole transcript. |
| Free text | a plain note, an email body, a paragraph | Use as-is. |

Distil to: what is being asked or proposed, why (the trigger or pain), and any owner, area, or date
hints the source contains. Do not invent detail the source lacks: missing fields get surfaced in the
proposal, never silently filled.

### N2: Capture provenance (first-class)

Provenance must survive into every downstream artifact (the Jira description footer, the migration
record, the outcome post). Capture two distinct roles plus the source pointer; they are often
different people:

- **Originator**: who actually raised the idea or request (the author of the pasted message, the
  person who proposed it in the meeting). This becomes the "raised by <originator>" attribution in
  the description. If the source has no identifiable author, fall back to the operator and flag it.
- **Operator**: whoever ran the intake. Always recorded as `filed by <user> on <date>`. Never
  impersonate the originator via the Jira reporter field; reporter stays whatever the API sets, and
  the attribution lives in the prose.
- **Source pointer**: the permalink verbatim, `from meeting <name> <date>`, or `pasted free text`.

Example: an idea Sam files from a transcript where Robin proposed it reads "raised by Robin,
from the weekly sync <date>, filed by Sam".

### N3: Multi-item split & selection

Most single messages are one item: skip this step. But a transcript (and occasionally a long thread)
routinely contains several distinct asks, some worth filing, many not. Do not auto-decide which count:

1. Enumerate every distinct actionable item with a short title, a tentative type (ticket / epic /
   idea / migration), and a one-line "why". Number them.
2. Present the list and ask the operator to select ("take 1, 3, 5" / "all except 2"). This is a
   selection checkpoint before any classification effort or write plan: discussion, decisions already
   taken, and chatter get dropped here by the operator, not guessed away by the skill.
3. Every selected item inherits the same provenance, plus a pointer to where in the transcript it
   came from if useful.

### N4: Classify the intent

Judge the distilled statement directly against the ontology (LLM judgement over the heuristics in
`team_ontology_page`, no rules engine). Per selected item, produce the list below. Without a
`team_ontology_page`, skip items 3 to 5 (and the idea shape's `axis::` candidate) and say so in the
proposal.

1. **Destination type**: `ticket`, `epic`, `idea`, or `migration` (decision guide below).
2. **Project**: per `routing_rules` (a common split: project or initiative work likely active more
   than a week goes to the delivery project; short-lived ops, incident, or on-call work goes to the
   ops project). When genuinely ambiguous, ask rather than guess.
3. **Work-type**: exactly one `wt::` label from the ontology's priority-ordered heuristics. Rule of
   thumb: maintain/upgrade/automate is ops; building a new capability is project work; when torn,
   default to the project-work label and flag it for grooming. Honour any carve-outs your ontology
   defines (e.g. security work carrying an `area::` label instead of a `wt::`, or an epic-only label
   that must never be assigned at intake).
4. **Area**: an `area::` hint where the source makes the sub-team clear; optional.
5. **Priority**: whether this is the weekly-focus label (`prio::now`). Default no: priority-setting
   belongs to the weekly planning ritual, not intake. Intake may note something looks urgent.
6. **Migration?**: whether this is an org-wide change other teams must adopt. If yes, the destination
   is `migration` regardless of the other fields.
7. **Initial status**: backlog (the default: most incoming items are new and unrefined), ready/next
   (clearly ready, or incident-shaped, for visibility), or in progress (only if someone is already
   actively working it, e.g. logged after the fact). Classify the intent, not the status name: live
   workflow names are read at create time.
8. **Incident-shaped?**: if the input describes an active incident or urgent breakage, set an
   incident flag. Intake still files the ticket but flags loudly that it likely needs on-call or
   immediate attention and is not a substitute for paging. Intake never pages or escalates.
9. **Confidence**: high / medium / low, with a one-line reason.

### Destination decision guide

| Signal in the distilled input | Destination |
|---|---|
| A concrete unit of work with a clear-ish owner or area | `ticket` |
| A large multi-ticket effort or theme | `epic` |
| A "good idea, not decided", no owner yet | `idea` (a Backlog Task) |
| An org-wide change other teams must adopt (a library deprecation, a forced upgrade, a platform adoption) | `migration` (hand off to migration-tracker) |

Idea vs ticket vs migration is the crux. A committed, ownable unit of work is a ticket or epic. A
"someday, no owner, not yet decided" is an idea: still just a Backlog Task, so the decision happens at
the commitment line (the move from Backlog into committed work, e.g. Next) and the weekly save-or-die review (the `backlog-resurfacing` skill) keeps the pool
honest. The cost of "wrongly" calling something an idea vs a ticket is therefore low. Something your
team would lead but other teams must do is a migration (the onboarding test: a single owner, a
deadline, expected teams).

### N5: Emit the proposal (no writes)

Present a proposal table before any routing: destination, summary, labels, priority, provenance,
confidence with a one-line reason. For a multi-item input, present one consolidated table, one row per
selected item, then approve the batch (or per-item) at a single gate.

The proposal rules (locked):

- **Never auto-file.** Always surface type + target + payload + confidence and wait.
- **Low confidence or multi-match: present the top 2 or 3 ranked options, don't guess.**
- **One item, one destination.** Don't fan one item across multiple homes; don't merge distinct items.
- **Flag missing fields, never invent them.**
- **Incidents are flagged, not buried.** The proposal (and the outcome post) carries a prominent
  "incident-shaped: likely needs on-call attention; this ticket is not a substitute for paging" line
  and proposes a ready/next initial status.
- Empty or pure-noise input: say there is nothing actionable and stop.
- Input that is already a Jira key: point at the existing issue instead of creating a duplicate.

## Capability 2: Route to Jira

File each approved `ticket`, `epic`, or `idea` in one of three issue shapes (an idea is always a
Backlog Task with no due date), with a written description carrying the originator and a provenance
footer, a proposed initial status mapped to live transitions at execution time, and a read-only
pre-write dedup search whose likely matches go into the plan. Writes are sequential and every new key
is verify-read before success is claimed. Read-only over other teams' boards: never create there.

**Full procedure:** [references/jira-routing.md](references/jira-routing.md#capability-2-route-to-jira).

## Capability 3: Route to migration

1. **Confirm it is really a migration.** The test: does it have (or warrant) a single owner, a
   deadline, and a set of expected teams? "We should maybe deprecate X someday" with no owner and no
   intent to commit is an idea: file it as a Backlog Task instead of onboarding a phantom migration.
2. **Extract what intake can**, so the hand-off arrives pre-filled: name, proposed slug
   (`migration::<kebab>`), owner (+ backup if named), soft and hard deadlines (flag if missing),
   expected teams the input names, links, and the provenance.
3. **Hand off.** Intake performs no writes of its own on this branch: on approval, invoke the
   `migration-tracker` skill's onboard capability with the extracted fields. It owns the epic,
   register, and detail-page writes, runs its own dedup (refusing to double-onboard a slug) and its
   own approval gate. Don't re-check or re-implement any of that here; duplicated logic drifts.
4. **Graceful degradation.** If migration-tracker is unavailable in the session, do not improvise the
   onboarding. File the item as a Backlog idea with a clear line that it is a migration candidate to
   onboard once the skill is available, and say so.

## Capability 4: Announce the outcome

After a successful, verified write, create a one-line outcome post to `team_channel` as a Slack draft
(what + where + link + source; tickets and epics by default, incident-shaped items always and loudly,
Backlog ideas only on request, migrations never, since migration-tracker announces its own). The
manager sends it from Slack; if the draft tool is unavailable, the text in chat is the deliverable.
This skill never calls the send tool. No outcome post without a write.

**Full procedure:** [references/jira-routing.md](references/jira-routing.md#capability-4-announce-the-outcome).

## Usage

```
/intake "Stage deploys keep drifting from git; someone should reconcile them weekly"
/intake https://<workspace>.slack.com/archives/<channel>/p<ts>
/intake <paste a transcript>
/intake                        # the skill asks for the input to triage
```
