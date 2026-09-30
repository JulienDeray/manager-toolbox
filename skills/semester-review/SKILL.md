---
name: semester-review
description: |
  Prepares a performance review for a direct report: collects evidence (1:1
  history, source-forge activity, peer/manager/upward feedback), triangulates
  it, helps you tighten your own draft of the manager feedback, and builds the
  meeting prep page, with hard checkpoints so the manager stays in control and
  makes every judgment call. Use when preparing a semester or cycle review,
  writing review feedback, or processing peer feedback for a direct report.
---

# Semester review preparation

Extension of `prepare-1-1` for the review cycle. The goal is **assistance, not delegation**: the
skill gathers and organizes evidence, challenges the manager's draft, and prepares the meeting; the
manager makes every judgment call. The workflow is staged with **two hard checkpoints**; never run
past a checkpoint without an explicit reaction from the manager. It never writes or rates the
review: every judgment is the manager's, and it only shapes the manager's own words.

Write surface: local files only until the meeting (`people/reviews/{slug}-{cycle}/` and, after the
review, `people/{slug}.md`), one meeting prep page in the `notes_tool`, and a recap draft DM the
manager sends by hand. Nothing is sent to the person by the skill.

## Requirements

- `notes_tool` access is required: read the cycle's 1:1 pages, create the prep page, read the live
  notes afterwards.
- `source_forge` read access is optional (skipped for non-dev roles or unknown usernames).
- `chat_tool` read access is optional (the Phase 2.5 sweep); draft support is optional for the recap
  (without it, the recap is output in chat).

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root. Fields used: `reports_roster`, `notes_tool`,
`meeting_title_convention`, `review_cycle` (cadence plus the questions your company's review form
asks), `source_forge` (optional), `chat_tool`, `style_rules`. The person's private file is
`people/{slug}.md`: local-only, gitignored, never committed or shared.

## Methodology (the review "by the book")

Every output of this skill must honor these principles, whatever else changes:

1. **No surprises.** The review synthesizes what was already said in 1:1s during the cycle. Anything
   material that never appeared in a 1:1 is a feedback gap, not a review topic like any other: flag
   it during prep, deliver it carefully in the meeting (acknowledge it should have come up earlier),
   and note the lesson to give that category of feedback in-cycle next time.
2. **Evidence-based.** Every strength and growth area is anchored to concrete examples: merge
   requests, incidents, postmortems, dated 1:1 notes, specific deliveries. Each finding gets an
   **Evidence:** line. A claim without an example is an impression and goes to the thin-evidence
   list until the manager backs it or drops it.
3. **Triangulated.** Themes confirmed by 2+ independent sources (manager's observations, peer
   feedback, 1:1 history, forge data, cross-mentions) carry weight. Single-source impressions are
   labeled as such: they can still be raised, but framed as an observation to explore, not a
   conclusion.
4. **Continuity.** Every review explicitly checks progress against the previous cycle's improvement
   areas and goals: progressed / stalled / regressed, with evidence. Growth acknowledged is as
   important as gaps named; a positive arc deserves explicit recognition.
5. **Behavior plus impact framing.** Growth areas are framed as observable behavior and its impact
   (SBI: situation, behavior, impact), never as character traits. Each one gets a memorable one-line
   "framing given" the person can carry, recorded in the people file. Example: *"Pick the problem
   that unblocks the team, not just the one that's interesting to build."*
6. **Two-way and forward-looking.** The report self-reflects first, before any feedback lands.
   Upward feedback is explicitly invited ("Feedback for me?"). The meeting ends with 2-3 concrete
   goals for next cycle and the support the manager commits to for each.
7. **Peer-feedback hygiene.** Peer input is synthesized into themes and anonymized. Never read a
   peer quote verbatim to the report unless that peer explicitly agreed. Cross-mentions harvested
   from other people's 1:1s are background signal only: same rule, stricter, never verbatim, ever.
   The same applies to chat-tool evidence.

## Process context

Most company review forms revolve around two questions, answered in a short paragraph by the manager
(put your company's exact wording in `review_cycle`):

1. What are some things X does well?
2. How could X improve?

The report typically gives the same feedback upward about the manager. For ICs, the manager also
collects peer feedback (same two questions). For managers reporting to you, input typically comes
from their team and cross-team partners instead.

## Workflow

### Phase 0: Resolve person and bootstrap inputs

1. Resolve the person via the `reports_roster`: slug, IC or manager, forge username if known.
2. Check the feedback folder: `people/reviews/{slug}-{cycle}/` (e.g.
   `people/reviews/alex-<cycle>/`). Expected files, all optional:
   - `manager-draft.md`: your own draft answers to the two questions
   - `peer-*.md`: one file per peer (ICs); for managers, input from their reports and cross-team
     partners
   - `upward.md`: the report's feedback about you
3. If the folder is missing or empty, ask the manager to paste whatever feedback they have, and
   **save each pasted piece into the folder** so the run is reproducible. This folder is as private
   as `people/`: gitignored, never shared.
4. Missing inputs don't block: note what's absent (e.g. "no peer feedback yet; for an IC, consider
   requesting 2-3 before the review") and continue.
5. For managers, extend the evidence list with team-health signals: attrition, delivery trends,
   escalations, themes from your skip-levels with their reports.

### Phase 1: Collect evidence

1. Pull the last 6 months (or one review cycle) of the person's 1:1 pages from the notes tool, and
   their activity from the `source_forge` if a username is known.
2. Read `people/{slug}.md`: especially the previous review section, the patterns/themes table, and
   the `## Observations Log` (entries captured via `log-observation`). Log entries are first-class
   evidence for the triangulation table; `[open]` entries are candidates for the review
   conversation.
3. Read all feedback files from Phase 0.
4. **Cross-mention sweep:** search the person's name (and nickname variants) across your 1:1 pages
   with *other* people over the cycle. Teammates often drop organic feedback ("X helped me with...",
   "waiting on X for..."). Handle with care: treat as background signal attributed as "a teammate's
   1:1", anonymize in anything delivered to the report, never quote verbatim.

### Phase 2: Synthesize and challenge, then **CHECKPOINT 1**

Present the synthesis **in chat only** (nothing written to files or the notes tool yet):

1. **Triangulation table**: theme by source (manager draft / peers / 1:1s / forge / cross-mentions),
   so corroborated themes stand apart from single-source impressions.
2. **Contradictions**: where sources disagree (e.g. peers praise something the manager flagged).
3. **Surprise check**: growth areas that never appeared in a 1:1, flagged per the no-surprises
   principle.
4. **Continuity check**: last cycle's improvement areas and goals: progressed / stalled / regressed,
   with evidence.
5. **Thin-evidence list**: claims that need the manager's judgment or a concrete example before
   they're usable.

**STOP.** Wait for the manager to confirm, correct, add context, or drop themes.

### Phase 2.5: Chat-tool sweep (optional, hypothesis-driven)

After Checkpoint 1, offer a sweep of your chat tool when themes are thin on evidence or a strength
or growth area needs concrete examples: 3-5 threads per hypothesis, **shared team channels only**
(never DMs or private channels), findings as dated behavioral examples and never quoted teammates,
then update the triangulation table before Phase 3.

**Full procedure:** [references/chat-sweep.md](references/chat-sweep.md).

### Phase 3: Shape the manager paragraph, then **CHECKPOINT 2**

1. If `manager-draft.md` exists, **refine it**: preserve the manager's voice and judgments; tighten
   wording, anchor claims to evidence from the confirmed synthesis. Otherwise ask the manager for
   their own bullets first (a few words per point is enough) and shape those; never originate the
   judgments.
2. Shape them into two answers, ~100-150 words each: *does well* / *could improve*.
3. **STOP.** Iterate until the manager approves the wording.
4. Save the approved version to `people/reviews/{slug}-{cycle}/manager-final.md`:

```markdown
# Manager feedback: {Full Name}, {cycle}

## What are some things {FirstName} does well?

{~100-150 words. Evidence-anchored, the manager's voice.}

## How could {FirstName} improve?

{~100-150 words. Behavior plus impact, forward-looking.}
```

### Phase 4: Meeting prep page

The output looks like a **normal 1:1 page, directed towards feedback**: same divider structure, same
opener bullets and live-notes area, but titled `Semester review - {FirstName} - {YYYY-MM-DD}` so
reviews stay findable among regular 1:1s. Prep notes are **terse `full` style**: one line per point,
fragments over prose, no run sheet.

Page body below the opener bullets and a `**Semester review: {YYYY-MM-DD}**` title plus live-notes
space:

```markdown
## 📋 Prep Notes (AI-generated)

### Guide

**Them first**
- "How was your semester?" (let them talk)

**👍 Keep doing**
- {One line per strength. Evidence in 2-3 words.}

**👈 Improve**
- {One line per growth area plus the one-line framing in italics}

**🎯 Forward**
- {2-3 goal proposals}
- What support do they need from me?

**Close**
- "Feedback for me?"

### Key Themes from the Semester

**1. {Theme}**
{2-3 sentences max: dated evidence, why it matters for this review.}
```

Create the page in the notes tool, same mechanics as `prepare-1-1` (attendee, date, status Not
started).

**Sensitive prep stays out of the notes tool**: anticipated reactions, verbatim peer quotes, the
triangulation table, and the full synthesis live in chat and `people/reviews/{slug}-{cycle}/` only.
The meeting page may end up shared or seen; treat it as such.

### Phase 5: After the review (run on request)

Read the live notes first, record the review **as delivered** in `people/{slug}.md` (prior review
moved down, incorporated observations pruned), optionally sweep capture markers, ask the two gap
questions, then draft the recap as a DM to the person that is **never sent** by the skill (room-only
content: nothing from peer feedback, cross-mentions, chat sweeps, or private prep), mark the meeting
Done, and point at `create-pdp`.

**Full procedure:** [references/after-review.md](references/after-review.md).

## Usage

```
/semester-review {Person} [on {Date}]
```

Date is the review meeting date (optional; ask if missing and a meeting page is needed). Re-run on
request after the meeting for Phase 5.

## Edge cases

- **No previous review**: skip the continuity check; state it explicitly ("first review under this
  manager").
- **No people file**: create `people/{slug}.md` (Basic Info stub plus Observations Log) as part of
  Phase 5.
- **No forge username / non-dev role**: skip forge analysis; lean harder on 1:1s, feedback, and
  cross-mentions.
- **Review cancelled or moved**: the prep survives in `people/reviews/{slug}-{cycle}/`; re-running
  the skill later should reuse it and only refresh the 1:1 scan.
