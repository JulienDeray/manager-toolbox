---
name: standup-processor
description: |
  Runs the team's daily stand-up wrap-up. Turns a stand-up transcript into Jira comments and new
  tickets, a Slack wrap-up draft, follow-up suggestions, and a stand-up format rating in chat.
  It has no approval step: it writes prefixed context comments, clear-case new tickets, and
  append-only hygiene edits directly, reports everything ambiguous on a "Not written" list, and
  leaves the Slack wrap-up as a draft it never sends. A `dry-run` arg writes nothing.
  Use when processing the daily stand-up transcript, or when the user mentions daily stand-up
  notes or updates.
---

# Daily Stand-up Processor

The after-bookend of your team's daily stand-up. The stand-up itself stays a human-led ritual;
this skill turns what was said into Jira updates, a Slack wrap-up, follow-up suggestions, and a
rating of how well the stand-up followed the team's own format rules, shown in chat at the end.

This is a **Jira + Slack** routine (no Confluence writes). It is **fire-without-gate**: it writes
a small set of factual Jira records directly, leaves the Slack wrap-up as a draft it never sends,
and reports everything else on a "Not written" list (see [Write classes](#write-classes)). The
boundary is the same whether you invoke it by hand or put it on a schedule. Run `dry-run` first
to see exactly what it would write.

## Requirements

- **Jira MCP tools**: get issue, create issue, add comment, get remote issue links. Bulk JQL
  searches should go through the Jira REST API rather than the MCP search tool (see the
  truncation gotcha in [references/jira-reconciliation.md](references/jira-reconciliation.md))
  (helper script recommended; see the README caveats).
- **Slack MCP tools**: the message-draft tool (this skill never calls the send tool).
- **A transcript source** (soft dependency): a Drive/file-search MCP is used only when no
  transcript is pasted (Step 0a). If it is unavailable, fall back to asking the operator to paste
  the transcript rather than stopping the run.

If a required tool is missing, say so and stop rather than working around it.

## Step 0: load team context

Read **`TEAM_CONTEXT.md`** at the repo root before doing anything. This skill needs:

- `jira_project_keys` and `jira_site_url` (for browse links)
- `team_channel` (the team's Slack channel ID)
- `team_roster` (name to Slack ID, name to Jira account ID, sub-area) and `manager_slack_id`
  (how "the manager" is resolved)
- `standup_comment_prefix` (e.g. `[STANDUP DD-MM-YYYY]`) and `date_format`
- `team_ontology_page` (your label taxonomy: work-type, area, service labels) if you maintain one
- `direction_page` (the page naming your team's current direction and active direction labels,
  e.g. an `axis::` label set) if you maintain one
- `team_agreement_page` (optional): your stand-up format rules; the rating's dimensions 1-4
  follow it when present.
- `transcript_source` (where meeting transcripts land, e.g. Google Meet auto-transcripts in
  Drive, and the meeting title pattern)

Never hardcode these values; `TEAM_CONTEXT.md` is the living config. If a run discovers new
durable information (a corrected name, a new channel), list the correction in the run report; the
operator applies it. If a required field is missing, stop and say which one; do not proceed with
partial configuration. The ontology and direction pages are only required if the corresponding
checks (label hygiene, direction linkage) are in use; without them, skip those checks and say so.

## Step 0a: locate the transcript

- **If a transcript was passed as an argument, use it as-is.** Never override a pasted
  transcript with a search: an explicit paste is assumed intentional (e.g. an edited version).
- **If no transcript was passed**, search your transcript source (e.g. Google Drive) for a doc
  matching the recurring meeting's title **and** today's date. Watch date formats: Drive/Meet
  titles often render dates as `YYYY/MM/DD` even if your team writes `DD-MM-YYYY`. Match on a
  loose `title contains` fragment rather than an exact string (meeting titles drift), and if more
  than one plausible match comes back, list them and ask rather than guessing.
- **If a match is found but the doc is empty or notes-only**: an auto-transcript doc can be
  completely empty for 30+ minutes after the meeting, and an AI notes doc (e.g. Gemini) can be
  populated but carry no verbatim transcript section. Neither means "no transcript": say what
  was found and offer to wait-and-retry or take a pasted transcript. A notes-only doc may still
  be usable; flag the missing verbatim transcript and let the operator decide.
- **If no match is found** (meeting skipped, recorder not started, title drifted): say so and
  ask the operator to paste the transcript. Never fail the whole run over a missed auto-fetch.
- In a scheduled run with no transcript found, or more than one candidate, write nothing and say
  so in the run report.
- Once found, fetch the content and treat it exactly like a pasted transcript from here on.

**Transcript hygiene:** Follow the transcript hygiene conventions in `TEAM_CONTEXT.md`. Proposed
glossary rows go in the run report; a row is appended only on the operator's confirmation in the
same session, never on a scheduled run. A line whose meaning depends on an undecoded garble
produces no write: it goes on the Not written list.

## Tone of voice: terse by default

This skill writes terse by default. Two levels:

- **`lite`**: drop filler and hedging, keep articles and full sentences. For durable prose.
- **`full`**: also drop articles; sentence fragments OK. For ephemeral output read once.

**Always-exact rule, at either level:** numbers, Jira keys, `<@mentions>`/`<#channels>`, dates,
and links are never abbreviated or paraphrased. Terse trims words, not facts.

| Output | Level |
|--------|-------|
| New-ticket description (Cap. 1) | `lite` (durable Jira prose) |
| Context comment on a ticket (Cap. 1) | `full` (a daily working note, not documentation) |
| Internal Slack wrap-up draft (Cap. 2) | `full` |
| External/stakeholder Slack draft (Cap. 2) | `lite` (readers lack team context) |
| Follow-up suggestions (Cap. 3) | `lite` |
| Rating output (Cap. 4) | `full` |

## Write classes

Each write class has one fixed behaviour, for a manual run and a scheduled run alike. There is no
gated mode to switch to and no trust level to reach; changing a row is an edit to this file.

| Write class | Behaviour |
|---|---|
| Context comment with the `standup_comment_prefix`, on an explicit key or a single-candidate match | Fires |
| New ticket, when no match exists AND the project is clear from the speaker's sub-area AND the speaker resolves in `team_roster` AND the board read was complete (R2) AND the line carries no undecoded garble | Fires |
| Hygiene edits in `ticket-hygiene`'s append-only classes (hand-off as `ticket-hygiene auto`) | Fires |
| Team-channel wrap-up | Slack draft, never sent by the skill |
| `--external` message | Slack draft if `stakeholder_channel` is configured, otherwise text in chat |
| Ambiguous matches (2+ candidates), unrecognised speakers, lines that depend on an undecoded garble | Not written: reported |
| New tickets with an unclear or contested project, or when the board read was partial | Not written: reported |
| Hygiene edits outside the append-only classes (returned by `ticket-hygiene`) | Not written: reported |
| Glossary rows and `TEAM_CONTEXT.md` corrections | Proposed in the run report; applied only on the operator's confirmation in the same session |
| Transitions (status changes) | Never |
| Rating (Capability 4) | Chat only |

When in doubt, the "Not written" list is the correct destination, not a best guess.

## Run flow

1. **Gather + analyse (read-only):** pull Jira context, parse the transcript, match updates, build
   every write, hygiene candidate, and draft.
2. **Write in sequence:** comments, then new tickets, then `ticket-hygiene auto`, one at a time.
3. **Verify-read every write.** Parallel Jira responses can come back shuffled.
4. **Slack draft:** create the wrap-up draft.
5. **Run report:** everything written (with verify results), the "Not written" list with a
   one-line reason per item, the follow-up suggestions, the rating. The report plus the draft in
   Slack is the run's human checkpoint.

**`dry-run`:** stop after step 1 and print the run report as it would be (writes marked "would
write"), the draft, the follow-ups, and the rating, writing nothing to Jira or Slack. The hygiene
hand-off runs as `ticket-hygiene auto dry-run`. Use it for your first runs, before scheduling the
skill, after any roster, ontology, or glossary change, and as a periodic spot-check.

## Slack handoff: a human sends

Team-facing Slack messages (the wrap-up, the optional external draft) are created as real Slack
drafts in the target channel and never sent by this skill; the run report says where each draft
sits. If the draft tool is unavailable, the text in the run report is the deliverable; never call
the send tool for these. The rating never touches Slack.

## Capability 1: Jira reconciliation

Parse the transcript per team member; match explicit (`PROJ-101`) and implicit ticket references,
blockers, and completions against a standup-scoped Jira query. Build **new tickets** for work
with no matching ticket, **context comments** (with the `standup_comment_prefix`) where a
ticket's Jira state doesn't reflect what was said, and, for discussed tickets only,
**description updates** (a durable fact the discussion established that the description misses),
**label fixes** (missing or wrong labels per your ontology, no-clobber), and **direction-linkage
flags** (a discussed ticket whose epic carries no active direction label). Write what the Write
classes table marks as firing; report the rest on the "Not written" list.

**Full procedure:** [references/jira-reconciliation.md](references/jira-reconciliation.md).

## Capability 2: Slack summary

### Internal wrap-up (always)

Draft an epic-level wrap-up for the team channel (`team_channel` in TEAM_CONTEXT.md) in three
sections (what's moving, blockers / needs, new tickets & comments), with a soft cap per section.
Create it via the Slack draft tool: a real Slack draft, not just a preview in chat. The skill
never sends it. A catch-up run covering several days builds one combined draft (Slack keeps only
one draft per channel).

**Full procedure:** [references/slack-wrapup.md](references/slack-wrapup.md).

### External/stakeholder message (opt-in only)

Only build this if the operator passed `--external` when invoking the skill. If not passed, do not
mention this capability at all (no "want an external message too?" prompt). When requested:

- High-level only: no internal ticket numbers, no blocker detail that isn't stakeholder-relevant.
- Terse `lite` style; same draft mechanism; never sent by the skill.
- Target channel: ask the operator, unless TEAM_CONTEXT.md records a standing
  `stakeholder_channel`.

After creating drafts, report which channels got one ("Draft created in the team channel:
review and send from Slack"). Never claim something was sent when only a draft was created.

## Capability 3: follow-up action suggestions

After the Jira and Slack work, scan the transcript for signals beyond the standard updates and
list **up to 5** optional suggestions, prioritised by impact; never pad to hit a count. Order:
risk flags first, then blockers/escalations, then documentation, then housekeeping.

**Full procedure:** [references/follow-ups.md](references/follow-ups.md) (signal categories,
detection cues, suggestion template).

Suggestions are listed in the run report only; this skill executes none of them. If the operator
dismisses a category repeatedly across runs, reduce that category's frequency going forward
rather than re-litigating it each run.

## Capability 4: stand-up rating + improvement suggestions

Score the transcript against a 6-dimension rubric (four from your team's own stand-up format
rules, two from stand-up-discipline research) and generate **up to 3** improvement suggestions,
a ceiling, not a target. Deliver the result **in chat, as the final section of the run's closing
message**, after every other capability has finished (Jira writes executed and verified, Slack
drafts handed off), so the operator reads it once everything is done.

**No Slack involvement at all.** A self-DM to the operator triggers no Slack notification and
goes unread. Not posted to any channel, not written to Confluence, not tracked run-over-run.

**Full procedure:** [references/standup-rating.md](references/standup-rating.md).

## Usage

```
/standup-processor [paste transcript here]
/standup-processor --external [paste transcript here]
/standup-processor            (searches the transcript source for today's transcript first)
/standup-processor dry-run    (composes and prints everything, writes nothing to Jira or Slack)
```
