---
name: pull
description: |
  Pulls from your work knowledge base (`notes/`) when you have free time or are
  looking for what to pick up next: surfaces 3-5 open ideas/questions/
  observations with a why-now rationale, and does light zettelkasten gardening
  (flags rotting notes, proposes links between existing notes). Use when asking
  "what should I work on", "pull an idea", "what's in my pool", or to review or
  garden your notes.
---

# Pull: retrieve and garden the knowledge base

Counterpart of `intake-for-me`: capture fills `notes/`, `pull` draws from it. Read-only by default;
the only writes are link additions and status flips the manager explicitly ratifies, all in local
`notes/` files.

## Requirements

- A local `notes/` directory at the repo root (filled by `intake-for-me`). No external tools are
  needed.

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root. Fields used: `reports_roster` (slugs in wikilinks),
`team_intake_pipeline` (optional: the handoff nudge). The pool lives in `notes/` at the repo root.

## Workflow

### 1. Read the pool

Read every `notes/*.md` (skip `README.md`), parse frontmatter. The pool is the `status: open` notes.
If a context was given, filter and weight by it: audience ("for the team" matches `audience: team`),
energy ("thinking mood" favors `question`s; "2h free" favors actionable `idea`s), topic tags.

### 2. Menu: 3-5 candidates

One line each: **title, type · audience · age**, then a one-line **why now**. Why-now signals, best
first:

- **Linked deadline or event**: an upcoming meeting, review, or date the note ties to
- **Momentum**: related notes captured recently (the theme is live in your head)
- **Staleness**: open a long time; picking it up or killing it both beat letting it rot
- **Cluster weight**: several notes point at the same theme (check the wikilinks), so the theme is
  bigger than any single note suggests

Don't pad: if the pool has 2 open notes, show 2. If the pool is empty, say so and stop.

### 3. Gardening tail (always, briefly)

- **Rot check:** flag 1-2 open notes untouched for ~2+ months: "still alive? close / keep / act". On
  the manager's answer, flip `status: closed` (with a dated resolution line) or leave.
- **Link proposals:** propose 1-2 conceptual links between existing notes ("[[a]] and [[b]] circle
  the same theme, link them?"). Same rule as capture: **write only what the manager ratifies**; the
  linking is their thinking, not the skill's.
- **Handoff nudge:** any `handoff: team` note still open gets a one-line reminder. If
  `team_intake_pipeline` is this toolbox's `intake` skill, offer to invoke it with the note's
  normalised statement (it runs its own gate); otherwise repeat the note's paste-ready intake block.

Keep the tail to a few lines; this is a nudge, not a report.

## Usage

```
/pull [optional context: "2h free" | "thinking mood" | "for the team" | "something quick" | ...]
```

## Edge cases

- **"Next personal project"**: out of scope (work-only knowledge base); point at your personal notes
  instead.
- **The manager picks a candidate**: offer the natural next step (open the note, expand it, spin up
  the work, or create a task via `intake-for-me` routing). Don't start executing the idea unasked.
