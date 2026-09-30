---
name: planning-processor
description: |
  Runs the team's weekly planning/grooming wrap-up. Turns what was said in the weekly planning
  session (plus Jira / Confluence / label-ontology context) into a small set of organising
  actions: priority highlighting (a focus label plus a native-Priority bump), a Slack wrap-up
  draft with WIP/flow metrics, filing planning-raised ideas into the backlog, ticket-hygiene
  hand-offs on discussed tickets, and an optional terse output mode. Every write is previewed and
  approved first; the wrap-up is a draft the manager sends.
  The meta-goal is to reduce work-in-progress and keep priorities visible week over week.
  Use when running the weekly planning or grooming session, processing planning notes, or when
  the user mentions weekly planning, grooming, or setting this week's focus.
---

# Weekly Planning Processor

The **after-bookend** of the team's weekly planning/grooming ritual: the ritual gives the board
an explicit home for setting focus and reducing work-in-progress, and this skill captures its
outcomes. It is the sibling of `planning-agenda` (the *before* bookend).

This is a **Jira + Confluence + Slack** routine. Its outputs: a focus label (`prio::now` in the
examples; rename to taste) applied to the week's agreed focus items, a weekly WIP/flow-metrics
row on a Confluence page plus a terse Slack wrap-up draft, and planning-raised ideas filed into
the backlog. All of it is **preview-then-approve**: nothing is written to Jira, Confluence, or
Slack without showing the plan first.

**Scope**: the epic planning board (`epic_board_id` in TEAM_CONTEXT.md) is the primary surface;
the task/delivery board (`delivery_board_id`) is touched only lightly (a handful of key delivery
tickets get the focus label) and is **excluded from the WIP count**. If your epic board mixes
epics from more than one Jira project, two scope rules keep the tracked series honest:

- **The WIP/flow metric stays locked to one project's epics** (the one it was defined on). A
  tracked metric that quietly changes definition is worse than no metric; secondary-project
  epics are reported by the wrap-up's lane counts, never by the WIP number.
- **The focus label may land on any board epic, but the native-Priority bump only applies where
  the project's priority scheme is verified.** An unverified project gets the label and the
  planning comment; its Priority field is never touched.

## Requirements

- **Jira MCP tools**: get issue, edit issue, add comment.
- **Confluence MCP tools**: get page, create page, update page.
- **Slack MCP tools**: the draft tool (the wrap-up delivery).
- **Plan mode** (or an equivalent explicit approval step) for the write gate.
- **Bulk JQL searches go through the Jira REST API**, not the MCP search tool: MCP search
  results truncate silently (a hard row cap on some implementations, overflow files on big
  sets), and both of this skill's searches are over that threshold: the epic WIP query (~15-30
  results) and the focus-label holder query (~6-8). A truncated holder query leaves last week's
  labels uncleared; a truncated epic query computes WIP from a fraction of the board (helper
  script recommended; see the README caveats). Single-issue reads stay on the MCP get-issue tool, where truncation can't bite.
- **A transcript source** (soft): a Drive/file-search MCP for Step 0a.

If a required tool is missing, say so and stop rather than working around it.

## Step 0: load team context

Read **`TEAM_CONTEXT.md`** at the repo root. This skill needs:

- `jira_project_keys`, `jira_site_url`, `epic_board_id`, `delivery_board_id`
- `team_channel` and `team_roster`
- `planning_comment_prefix` (e.g. `[PLANNING DD-MM-YYYY]`) and `date_format`
- `wip_metrics_page` (the Confluence page carrying the weekly metrics table)
- `team_ontology_page` (focus-label rules, status mapping, label axes) and
  `team_agreement_page` (board model, WIP limits), if you maintain them
- `transcript_source` (where the planning session's notes/transcript doc lands)

Missing required fields are a hard stop; say which one. If a run discovers new durable
information, propose updating `TEAM_CONTEXT.md`: it is the living config, maintained in place.

## Step 0a: planning notes (when invoked without notes)

If no planning notes were passed in, **search your transcript source before asking**: a recorded
meeting tool (e.g. Google Meet with Gemini notes) usually drops a notes-plus-transcript doc
shortly after the session ends. Match **loosely** on the meeting title plus the planning date
(titles drift), and read the whole doc: the AI summary/decisions block **and** the raw
transcript. This is a **soft** dependency: if the source is unavailable or no doc matches, fall
back to gathering the board state and asking the user for the week's focus decisions.

When invoked by `stepback-processor`, use the notes it passes: skip the transcript search, and
skip garble proposals (the caller already confirmed them).

**Transcript hygiene:** follow the transcript hygiene conventions in `TEAM_CONTEXT.md`, and
decode before attributing anything (a misattributed focus item is exactly how the wrong ticket
gets bumped). New glossary rows are proposed in the plan and appended only once confirmed.

## Flow: plan, approve, execute, verify

1. **Gather + analyse (read-only):** enter plan mode, pull the Jira/Confluence/ontology context,
   parse the notes, and build a concrete plan of every proposed write (label changes, Priority
   bumps, the metrics row, the ideas heading to the backlog).
2. **Approve + execute:** present the plan. Only after explicit approval, execute the writes.

**Verify-read after every write.** Parallel Jira edit responses can come back shuffled: after
applying label/Priority changes, re-read the affected issues to confirm the writes landed as
intended before reporting success.

## Slack handoff: draft in channel, the manager sends by hand

Team-facing messages are **never posted by this skill**. Deliver the wrap-up as a **Slack draft
attached to the channel** (the draft tool, `team_channel`), created automatically once the plan
is approved, with no extra confirm step: the manager reviews and edits the draft in Slack's
"Drafts & sent" and sends it by hand. Slack allows only one attached draft per channel; if one
already exists, report it and hand over the text instead of failing. If the draft tool is
unavailable, the text in chat is the deliverable; never call the send tool for it.

## Capability 1: priority highlighting

Clear last week's focus label, apply it to the agreed focus epics plus key delivery tickets, and
bump native Priority on the **top 1-3** focus items. Labels are read-modify-write (never clobber
the other labels on an issue); writes are sequential; every change is previewed before execution
and confirmed by a verify-read after.

Then post a planning comment (with the `planning_comment_prefix`) on each focus item: **current
stage** plus **what's expected this week**, drawn from the planning notes, so the board carries
the week's intent.

**Full procedure:** [references/priority-highlighting.md](references/priority-highlighting.md).

## Capability 2: Slack wrap-up + WIP/flow metrics

Compute the weekly WIP snapshot (in-progress epics in the tracked project, sum and average of
in-progress age, top-N oldest offenders, and optional board-lane counts), append a row to the
WIP metrics Confluence page, and deliver a terse wrap-up as a Slack draft.

**Full procedure:** [references/slack-wrapup-metrics.md](references/slack-wrapup-metrics.md):
the locked metric definitions, the changelog-based age rule and its fallback, the Confluence row
append with the storage-format caveat, the wrap-up shape, and the verify-reads.

## Capability 3: ideas to the backlog

Ideas raised during planning that are **not this week's work** ("we should look at X someday",
deferred decisions, no owner yet) are filed into the backlog as idea-shaped Tasks, so they
survive the meeting without becoming commitments. Collect them (title + context + raiser + the
note line), and:

- If the `intake` skill is installed, **invoke it** with the collected ideas
  (invoke, don't inline: it runs its own classification, dedup, gate, and verify-read). The
  raiser attribution and a "from planning <date>" provenance line travel in the hand-off.
- Otherwise, propose the creates in this skill's own plan: one backlog Task per idea, terse
  `lite` description carrying the context, the raiser, and the provenance line. Same
  preview-then-approve gate as everything else.

If neither path is possible in the session, do not improvise: list the ideas (title + raiser) in
the wrap-up and say they await filing.

**External scope:** when an outside ask lands mid-planning, show the operator the WIP trade-off
(what would drop to take it on). Advice for the conversation, not a team rule or a wrap-up line.

## Capability 4: ticket hygiene

When the session's notes **discuss a ticket** whose description misses a material point the
conversation established (or has no description at all), or whose labels don't match the
ontology, collect the candidates (ticket key + the note line that prompted each) and hand them
to `ticket-hygiene` if it is installed (it runs its own gate, shows before-and-after diffs,
executes, verify-reads). Default-lane epics the session described as collaborations (M3b) are
handed over the same way.

**Fallback:** without `ticket-hygiene`, either propose the edits as an explicit, clearly-separated
section of this skill's own plan (before-and-after diffs; description edits additive-only; label
fixes no-clobber), or list the candidates (key + reason) in the wrap-up for a later pass. Never
silently rewrite a ticket.

## Tone: terse levels

An optional **toggle** (off by default, opt-in per invocation) that renders the routine's
human-facing text in a short, declarative style. Two levels:

- **`lite`**: drop filler and hedging, keep full sentences (the default when terse is requested).
- **`full`**: also drop articles; fragments OK.

Output durability sets the cap: `full` is for the **ephemeral** Slack wrap-up and in-chat
summaries; durable/structured outputs (Confluence prose, Jira prose fields, plan-preview tables)
cap at `lite`. It is a **presentation layer only**: never the data, metric definitions, label or
Priority *values*, verify-reads, or the approval gate. Numbers, all Jira keys, Slack
`<@mentions>` and `<#channels>`, and links are always reproduced verbatim; terse output trims
words, not facts. Ask if the requested level is ambiguous.

## Helper computations

Two fiddly parts benefit from a deterministic helper (helper script recommended; see the README
caveats); if computing by hand, follow the rules exactly as written in the references:

- **Epic in-progress age** from Jira changelog payloads (the active-entry rule and its fallback):
  [references/slack-wrapup-metrics.md](references/slack-wrapup-metrics.md), M2.
- **Confluence table-row splice** (insert one row newest-at-top, every other byte untouched):
  same file, M4. Hand-editing a Confluence storage body is the fragile part; if you must, edit
  only the bytes of the new row and diff before writing.

## Usage

```
/planning-processor [paste planning-session notes here]
/planning-processor          (searches the transcript source; else gathers board state and asks)
```
