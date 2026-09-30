---
name: intake-for-me
description: |
  Captures things you want to remember for yourself as a manager (ideas,
  questions to chew on, observations, project seeds, the occasional concrete
  TODO) into your work knowledge base (`notes/`), your task system, or a handoff
  to a team pipeline. Works as a one-off (`/intake-for-me {blob}`) and as a
  sweep that picks up inline `/intake-for-me`, `/intake-team`, and `/intake-org`
  markers you drop in meeting notes during 1:1s and reviews. Use when noting
  something for yourself ("remind me", "I should think about", "idea:"), or to
  sweep meeting pages for markers.
---

# Intake for me: personal knowledge base capture

Sibling of `log-observation`, pointed at you instead of a report. Same philosophy: **no hard
checkpoint**. Distill, route, write, then echo a recap so you can correct it. The whole exchange
must cost less than a sticky note.

Destinations (see routing below): atomic notes in `notes/` at the repo root, dated tasks in your
`task_system`, or a handoff to a separate team pipeline. Notes are consumed by `pull`; keep them
atomic and linked.

Write surface: local `notes/` files, dated tasks in your `task_system`, the optional `history_file`
(via `log-observation`), and, in sweeps, a targeted `✅ intaken` rewrite of each processed marker
token on the meeting page. Team-bound items are handed off, never filed around the team pipeline's
own gate.

## Requirements

- A local `notes/` directory at the repo root.
- `task_system` access for dated tasks.
- `notes_tool` access for sweeps over meeting pages (the one-off mode doesn't need it).

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root. Fields used: `reports_roster` (for people wikilinks),
`notes_tool` (for sweeps over meeting pages), `task_system` (where dated personal tasks go, e.g. a
tasks database in your notes tool), `team_intake_pipeline` (optional: a separate team repo or
process that team-aimed items get handed to), `history_file` (optional).

Two modes, detected from the argument (see Usage): a one-off blob (Mode A) or a sweep (Mode B).

## Mode A: one-off blob

1. **Fetch any links**: same rules as `log-observation` (fetched content is context for distilling,
   never dumped into the note).
2. **Route and write** (rules below).
3. **Echo the recap.**

## Mode B: sweep

1. **Resolve the target pages:**
   - Page URL: that page.
   - Person name: their recent meeting pages in the notes tool.
   - `recent`: meeting pages with status Done and date within the last 14 days.
2. **Scan each page** for `/intake-for-me`, `/intake-team`, and `/intake-org` markers.
   - The item is the bullet carrying the marker **plus its child bullets and immediate context**
     (use judgment: the marker tags a thought, not a single line). Capture your stance when the
     surrounding notes give one.
   - **Skip** markers already rewritten to `✅ intaken`, and anything whose source (page plus anchor
     text) already appears in an existing note's `source` frontmatter.
3. **Route and write** each item (rules below).
4. **Rewrite each processed marker in place** to `✅ intaken`: a targeted text replacement of the
   marker token only. **Never rewrite whole blocks of live meeting notes.** If the rewrite fails,
   say so in the recap and move on; the `source` dedup prevents double-capture on the next sweep.
5. **Echo the recap.**

When invoked from another skill's post-meeting flow (`semester-review` Phase 5, `wrap-1-1` Step 6),
the sweep runs as a background sub-agent: it is autonomous end-to-end and needs no mid-run input.

## Routing rules (both modes)

Evaluate in order:

### 1. Concrete and time-windowed: a task

A specific, scoped action with an actual or inferable time window ("review Sam's promo doc, draft
due <date>") becomes a dated task in the `task_system`:

- Short imperative title
- Date inferred from content (e.g. a few days before a stated deadline); flag the inference in the
  recap. No inferable window: it probably isn't task material; treat it as a note.

**Task only, no note**, unless the task is an instance of a theme worth tracking, in which case the
theme note is created or extended and mentions the task.

### 2. Team handoff

An `/intake-team` marker, or a blob clearly aimed at your team's own intake pipeline (a separate
repo or process with its own approval gates, which must not be bypassed):

- Write an atomic note with `handoff: team`, `status: open`.
- If `team_intake_pipeline` is this toolbox's `intake` skill, offer to invoke it with the normalised
  statement (it runs its own gate); otherwise emit the paste-ready block below. A background sweep
  (see Mode B) always emits the block.
- The paste-ready block goes in the recap for the manager to run in the team pipeline's own session,
  e.g.:

  ````
  ```
  /intake "{normalized problem statement}. Source: {meeting page title + URL / provenance}"
  ```
  ````

- The note flips to `closed` when the manager reports it filed (same update path as below).

### 3. Org event: history file

An `/intake-org` marker, or a blob that is an org or relational event (a departure, arrival, team
move, reorg, a collaboration/praise/friction signal between people): route it through
`log-observation`'s org-event path into the `history_file` (same session is fine), with the meeting
page as source. **History only, no note**, unless the item also carries an idea or question for your
pool, in which case both.

### 4. Everything else: atomic note

One markdown file per note in `notes/`, frontmatter plus a short body:

- `type`: `idea` (potential project or do-later, including team initiatives) | `question` (thing to
  think about) | `observation` (noticed pattern worth remembering). When unsure, infer from tone;
  don't ask.
- `audience`: `me | team | org` (or your own finer-grained values); `tags`: free-form topical.
- `created`: ISO date (today unless the blob says otherwise); `source`: meeting page URL / chat link
  / `chat`; `status: open`.
- Body: 2-6 lines, facts plus your take at the time. Distillation rules from `log-observation`
  apply: behavior over interpretation, no verbatim teammate quotes, anonymize third parties unless
  naming them is the point.
- **Dedup first**: if an existing open note covers the same theme, extend it (dated addition to the
  body) instead of creating a near-duplicate.

## Linking (zettelkasten discipline)

- **Auto-write mechanical links only**: people mentioned become `[[{people-slug}]]`; the source
  meeting reference.
- **Conceptual links are proposed, never auto-written**: in the recap, list up to 2-3 existing notes
  this one echoes ("echoes [[on-call-rotation-fragility]], link?"). Write only the ones the manager
  ratifies. The linking is where their thinking happens: lower the lookup cost, don't do the
  thinking for them.

## Recap format

One table, then the proposals:

```
| Item | Destination | Links |
|------|-------------|-------|
| {short title} | notes/{slug}.md ({type}) | auto: [[x]] · proposed: [[y]]? |
| {short title} | task {date} (inferred from {...}) | none |
| {short title} | notes/{slug}.md + team handoff | auto: [[x]] |
```

Plus, when applicable: the paste-ready team-handoff block(s), and any failed marker rewrites.

## Update path (closing notes)

When a blob reads as a conclusion ("filed the access thing with the team", "dropped the planning
idea"): find the matching open note (match on theme), append a dated `**Resolution ({YYYY-MM-DD}):**
...` line to the body, flip `status: closed`. No match: treat as a new capture.

## Usage

```
/intake-for-me {blob of text: an idea, reminder, observation; optionally with links}
/intake-for-me sweep {meeting page URL | person name | "recent"}
```

A bare `sweep` means `sweep recent`.

## Edge cases

- **Marker with no adjacent content** (bare `/intake-for-me` on an empty line): ask what it was
  meant to capture; don't guess from the whole page.
- **One bullet carrying two markers**: both routes; the note serves `/intake-for-me`, the handoff
  block serves `/intake-team`.
- **Observation about a specific report** that belongs in their people file: that's
  `log-observation`'s job. Log it there instead (same session is fine) and say so in the recap.
- **Sensitive content**: same rule as `log-observation`; record the operational fact, not the
  private detail.
- **Personal-life items** (gift ideas, trips, recipes): out of scope; this is a work-only knowledge
  base. Point at your personal notes instead of writing a note here.
