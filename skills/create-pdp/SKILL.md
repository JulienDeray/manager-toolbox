---
name: create-pdp
description: |
  Creates and develops a personal development plan (PDP) for a direct report,
  as a doc shared with them. Stage A builds the skeleton doc right after the
  review (company/engineering/team goals filled in, main goal left for the
  person); Stage B, ~2-3 weeks later, proposes intermediate goals grounded in
  what was already discussed in the review and 1:1s. Use when creating a
  development plan, setting cycle goals with a report, or turning review
  outcomes into a development plan.
---

# Personal development plan (PDP)

Companion to `semester-review`: the review draws the line on the past cycle; the PDP points at the
next one. Once per review cycle, per report. The philosophy is the same, **assistance not
delegation**, and the material rule carries over from the review methodology: **PDP goals are never
invented**. Every proposed goal traces back to something already discussed (review, 1:1, previous
PDP); anything without provenance is explicitly flagged as new.

Write surface: one plan doc per person per cycle in your `pdp_storage` (created at Stage A; Stage B
outputs paste-ready text unless your docs integration can edit the doc and you accept), the
per-cycle `team_goals_file`, and the `## Development plan` section of the local `people/{slug}.md`.
It never touches sharing or permissions and sends nothing to the person.

## Requirements

- A docs integration for your `pdp_storage` (search, read, create). Without create access, the Stage
  A skeleton is output as paste-ready markdown instead.
- Local `people/{slug}.md` files and the `people/reviews/` folder written by `semester-review`.

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root. Fields used: `reports_roster`, `pdp_storage` (where plan
docs live, e.g. a per-person folder in your docs tool, plus the title convention), `team_goals_file`
(path convention for the per-cycle goals file, e.g. `people/goals/<cycle>.md`). The person's private
file is `people/{slug}.md`: local-only, gitignored, never committed or shared.

## Stage detection

The skill detects the stage itself: search the `pdp_storage` for a plan doc for the **upcoming
cycle**, using the title convention (default: `Personal Development Plan - {FirstName} - {cycle}`).
Try name variants (nicknames, disambiguated short forms).

- No doc yet: **Stage A** (create the skeleton)
- Doc exists: **Stage B** (intermediate goals)

## The PDP process

1. Right after the review, the manager creates the PDP doc with **their part** filled in: the
   company's goals, engineering's goals, the team's goals for the cycle.
2. The person gets ~2-3 weeks (until a next 1:1) to reflect and fill **their part**: "Your main
   goal", what drives them, who they want to become. Promotion, family time, money, a technology,
   visibility: anything, as long as it's theirs.
3. Then manager and person work **together** on the intersection: 3-4 **intermediate goals** that,
   once attained, move the person toward their goal *and* the company toward its goals. Each has a
   deliverable and a target date (the plan must stay Assessable).

## Stage A: Skeleton

### Phase A0: Resolve

1. Person from the `reports_roster`; slug as used in `people/`.
2. Cycle = the upcoming half (or your company's next review period).

### Phase A1: Gather

1. **Previous PDP:** search the storage by title (also try older spellings of the title and name
   variants). Read it and capture:
   - Its **folder**: each person has one; the new doc goes there.
   - Its **URL and title**: becomes the "Last PDP" link.
   - Its **goals**: report their status in chat (done / carried over / dropped, from the people file
     and review outcomes). This is continuity input for Stage B and for the manager's conversation;
     it does not go in the new doc.
   - No previous PDP: first cycle, say so, skip continuity, ask the manager where the doc should
     live.
2. **Cycle goals:** read the `team_goals_file` for this cycle. If missing, ask the manager for the
   company / engineering / team goals and **save the file**: it's written once and reused for every
   report's PDP that cycle. Format:

```markdown
# Cycle goals: {cycle}

## Company
- {short labeled lines, not paragraphs}

## Engineering
- {...}

## Team
- {...}
```

### Phase A2: **CHECKPOINT**

Render the complete doc content in chat, exactly as it will be created: title, Base Principles,
goals block, Last PDP link, empty "Your main goal", four empty intermediate-goal blocks. **STOP.**
Wait for the manager's approval or corrections.

"Your main goal" stays **empty**: the person reflects first, before anything from the manager (or
from the review) can anchor them. Review-derived material enters only at Stage B, and only as
proposals in chat.

### Phase A3: Create the doc

Create the doc in the `pdp_storage` (e.g. a Google Doc in the person's folder), from this skeleton:

```markdown
# Personal Development Plan - {FirstName} - {cycle}

## Base Principles

- Keep you and the business **Aligned**
  - Your goals help the business hit its targets
  - The business helps you hit your targets
- Makes us **Accountable**
  - I commit to helping you achieve your goal
  - You commit to try and achieve your goal
- It remains **Agile**
  - We review progress regularly
  - React to events rather than following the plan (if goals stop being relevant or the situation changes, it's okay to update the plan)
- It is **Assessable**
  - We should be able to measure progress and success at any time

## Company goals this cycle

{contents of the cycle goals file, rendered as short labeled lines}

## Last PDP

[{Previous PDP doc title}]({previous PDP URL})   (or `N/A` on a first cycle)

## Your main goal

## Intermediate goals

Goal:
Deliverable:
Target date:

Goal:
Deliverable:
Target date:

Goal:
Deliverable:
Target date:

Goal:
Deliverable:
Target date:
```

**Never touch sharing or permissions**: the manager shares the doc with the person themselves,
always. If your docs integration cannot create the doc, output the skeleton as paste-ready markdown
instead.

### Phase A4: Record and handoff

1. Add or update the `## Development plan: {cycle}` section in `people/{slug}.md` (doc link, status
   `awaiting main goal`).
2. Remind the manager: share the doc with the person; agree on a 1:1 ~2-3 weeks out by which "Your
   main goal" is filled; run `create-pdp {Person}` again then.

## Stage B: Intermediate goals

### Phase B0: Read the doc

Fetch the current-cycle PDP. If "Your main goal" is still empty, **stop**: tell the manager, and
offer a nudge line for the next 1:1 prep instead.

### Phase B1: Gather the "already discussed" material

1. `people/{slug}.md`: latest review section (Reactions & Agreed Goals, improvement areas with their
   framings, patterns/themes table).
2. `people/reviews/{slug}-{cycle}/manager-final.md`: the delivered paragraph.
3. Unfinished goals from the previous PDP (Phase A1 continuity notes).
4. Career-path context from 1:1s if relevant (e.g. a promotion-track initiative).

### Phase B2: Propose, then **CHECKPOINT**

Propose 3-4 intermediate goals at the intersection of the person's main goal, the cycle goals, and
the review material. For each:

```markdown
**Goal {n}: {short name}**
- Goal: {what}
- Deliverable: {the assessable artifact or outcome}
- Target date: {YYYY-MM or a milestone}
- Provenance: {agreed at the review | <date> 1:1 | carried from last PDP | ⚠️ new, not yet discussed}
```

A goal that never appeared anywhere is flagged **⚠️ new, not yet discussed** and framed as a
suggestion to raise, never as something agreed.

**STOP.** Iterate until the manager is happy with the set.

### Phase B3: Output for the doc

Output the approved intermediate-goals block as clean paste-ready markdown for the manager to put in
the doc (**no provenance tags**; those are prep-only):

```markdown
Goal: {what}
Deliverable: {...}
Target date: {...}
```

If your docs integration can edit the existing doc, offer to apply it instead; otherwise paste-ready
is the default. Offer to add a one-line pointer ("finalize PDP intermediate goals") to the prep
notes of the 1:1 where they'll be agreed.

### Phase B4: Record (after the goals are agreed)

Update `## Development plan: {cycle}` in `people/{slug}.md`: main goal, final intermediate goals,
status `active`. Future 1:1 preps and the next `semester-review` continuity check track progress
against this section.

```markdown
## Development plan: {cycle}

- Doc: [{title}]({doc URL})
- Status: {awaiting main goal | active | closed}
- Main goal: {once filled by the person}
- Intermediate goals:
  - 🎯 {goal}: {deliverable}, target {date}
```

## Guardrails

- **The doc is shared with the person.** Nothing sensitive goes in it: no peer quotes, no
  anticipated reactions, no assessment that wasn't already delivered in the review. Same rule as the
  meeting pages.
- **Provenance on every goal**, or the ⚠️ new flag. No invented goals presented as agreed.
- **Sharing is the manager's manual step**, always. The skill never modifies doc permissions.

## Usage

```
/create-pdp {Person}
```

Run it once right after the review (Stage A) and again once the person has filled "Your main goal"
(Stage B); the stage is detected from the doc.

## Edge cases

- **No review this cycle**: the PDP normally follows the review. Warn, and only proceed on the
  manager's say-so, using 1:1 history and the people file as material.
- **Person is a manager**: review artifacts are team-health and leadership shaped; goals lean on
  those rather than IC delivery items.
- **Mid-cycle update**: the plan is Agile; if goals stopped being relevant, rerun Stage B against
  the existing doc and re-propose. Output is again a paste-ready block.
- **New joiner, no review and no previous PDP**: proceed; the goals block and main-goal reflection
  still apply, and intermediate goals draw on onboarding conversations instead.
