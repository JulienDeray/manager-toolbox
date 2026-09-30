---
name: prepare-1-1
description: |
  Prepares a 1:1 meeting page in your notes tool before the meeting. Reads your
  private per-person knowledge base (open observations, agreed goals, development
  plan status, recent org events), analyzes the last few meetings for open and
  closed topics, and creates a prep page with a private briefing above a divider
  and suggested questions below it. Only the part below the divider is shared
  with the report. Use before a 1:1 or skip-level: "prep my 1:1 with Alex",
  "prepare tomorrow's 1:1s", or a pasted list of upcoming meetings.
---

# Prepare 1:1

Create a preparation page for each upcoming 1:1: a private pre-meeting briefing for you, plus
contextual suggested questions and open-topic follow-ups derived from meeting history.

Write surface: one new meeting page per 1:1 in your `notes_tool` (and, for a skip-level with no
people file, a minimal local `people/{slug}.md` stub). It reads everything else and sends nothing.

## Requirements

- `notes_tool` access is required: search and read past meeting pages, create the prep page.
- Local `people/{slug}.md` files and the optional `history_file` feed the briefing; missing ones are
  skipped (see the Step 2 edge cases).

## Step 0: load team context

1. Read `TEAM_CONTEXT.md` at the repo root. Fields this skill uses: `reports_roster`, `notes_tool`,
   `meeting_title_convention`, `opener_questions`, `history_file` (optional).
2. Resolve the person to their slug via the `reports_roster` (name, slug, role, relationship,
   active). Fallback convention: `{firstname}-{lastname}` kebab-case, using the name the person
   actually goes by.
3. Their private knowledge base is `people/{slug}.md`. These files hold sensitive management notes:
   they are local-only, must be gitignored, and must never be committed or shared.

## Page templates

The first divider on the page is the sharing boundary. Everything above it is the private briefing
(yours only); everything below it is the shareable part, the only part the report ever sees.
Knowledge-base content must never leak below the divider.

### Regular 1:1

```markdown
## 🔒 Pre-meeting briefing (private: everything below the divider is the shareable part)

**Open observations:**
- {date}: {title} ({⚠️/✅/ℹ️}), {one-line status}

**Agreed goals to check on:**
- 🎯 {goal from the latest review}

**Development plan:** {status / main goal}

**🕰 Since we last spoke:**
- {date}: {event title} ({type}), {one-line relevance for this 1:1}

---

{Your standing `opener_questions`, from TEAM_CONTEXT.md. Defaults:}
- What's on your mind?
- And what else?
- What's the real challenge here for you?
- What do you want?

**🚀 1:1 Summary: {YYYY-MM-DD} 🚀**

{SUGGESTED_QUESTIONS, placed directly here, no header}

{empty space for live notes}

---

## 📋 Prep Notes (AI-generated)

### Open Topics to Follow Up
{OPEN_TOPICS}

### Likely Closed Topics
{CLOSED_TOPICS}
```

Briefing rules: omit sub-sections with nothing to report; omit the whole briefing and its divider
only when there is nothing at all to brief (no people file, no history matches). The meeting page
may end up shared or seen; treat it as such, and keep anything more sensitive than the briefing
template's own fields (verbatim quotes, anticipated reactions) out of the notes tool entirely.

### Skip-level 1:1

Same divider structure. The briefing carries "Since we last spoke" events and any open observations
from `people/{slug}.md`. Below the divider, replace the opener questions with skip-level ones:

- What's working that we should keep or scale
- Friction points or blockers slowing delivery
- Improvement ideas (process, standards, tooling)
- What's your biggest challenge right now?
- Quick next steps (owners, timelines)

Then a `**Skip 1:1: {YYYY-MM-DD}**` title, suggested questions, a live-notes area, a `**Follow-up:
Action items**` stub, and the AI prep notes below a divider.

## Workflow

### Step 1: Parse the input

Extract person, date, and meeting type (regular or skip-level) per line. Multiple meetings can be
pasted at once for batch processing (see Batch runs). A skip-level is any meeting titled "Skip 1:1"
or flagged as such, or a person whose `reports_roster` relationship is `skip-level`.

### Step 2: Load the local knowledge base

Before touching the notes tool, pull what you already know.

Read `people/{slug}.md` and extract:
- `## Observations Log`: all `[open]` entries (heading format `### {YYYY-MM-DD}: {title} {⚠️|✅|ℹ️}
  [open]`), plus `[closed]` entries from the last ~4 weeks for recent context.
- The latest review section: agreed goals (🎯) and improvement areas.
- `## Patterns & Recurring Themes` if present.
- `## Development plan` status and main goal if present.

Read the `history_file` if TEAM_CONTEXT.md names one (a local log of dated org and relational
events: departures, arrivals, team moves, role changes, manager changes, reorgs, mentorship changes,
collaborations, praise, friction, bridges). Select events involving this person, their teams, or
(for skip-levels) their manager. Window: since your last meeting with them, widened to a minimum of
3 months and capped at 12. Render matches as the "Since we last spoke" block, one line each, most
relevant first; mark unconfirmed events as such. Relational events carry attribution in their source
field: private context that informs your questions, never quoted below the divider.

Edge cases:
- No people file + skip-level meeting: create a minimal stub first (`# {Full Name}`, a `## Basic
  Info` table with TBDs, `## Observations Log`; note `relationship: skip-level`). A skip-level
  person's observations live in their own file from then on, not their manager's.
- No people file + one-off meeting: skip the file-based parts; still check the history file.
- Empty log, no history: omit those blocks and carry on.

### Step 3: Search past meetings

Search your notes tool for the person's recent meeting pages using the `meeting_title_convention`
(e.g. "1:1 {FirstName}", "Skip 1:1 {FullName}"). If nothing matches, try variants: "1-1 {Name}",
"{Name} weekly", "{Name} meeting", and diacritic-free spellings.

### Step 4: Fetch meeting content

Retrieve the last 3 meetings per person. That is enough context without drowning the analysis.

### Step 5: Analyze topics

Open topics: ongoing projects, problems raised without resolution, action items not confirmed done,
questions not fully answered, themes recurring across meetings, development goals in progress.

Closed topics: items explicitly marked done, one-time events that passed, decisions made and
implemented, concerns later followed by expressed satisfaction, topics absent from recent meetings.

Judgment guidelines: discussed only in the oldest meeting, likely closed; appears across multiple
meetings, still open; action item with no follow-up, still open. When in doubt, keep it open: better
to ask than to miss.

### Step 6: Generate suggested questions

4-6 contextual questions that:
- Follow up on open topics naturally
- Don't repeat questions already asked recently
- Address concerns or challenges raised
- Check on wellbeing if relevant
- Are conversational, not interrogative
- Are informed by the briefing (open observations, agreed goals), phrased as natural conversation
  openers

Privacy rule (absolute): content from the Observations Log or reviews goes only in the briefing
above the divider. Below the divider, that knowledge may only shape which questions get asked,
phrased naturally, never quoting or revealing a logged concern or review finding.

Format the questions as bullets directly under the summary title, no header.

### Step 7: Create the page

Create the meeting page in your notes tool where your 1:1 pages live (`notes_tool` in
TEAM_CONTEXT.md), titled per `meeting_title_convention` (defaults: `1:1 {FirstName} - {YYYY-MM-DD}`,
`Skip 1:1 - {FullName} - {YYYY-MM-DD}`). Set the date, link the attendee, and set status to Not
started if your tool supports these.

## Batch runs

When the input contains more than one meeting, parse in the main session, then spawn one
general-purpose subagent per person, all in parallel. Each subagent reads this SKILL.md and runs
Steps 2-7 for its assigned meeting, creating the page itself, and returns only: the output-table
row, the key topics identified, and a one-line briefing recap. Rationale: each person's knowledge
base stays in its own isolated context, so nothing bleeds between people and a long batch can't
degrade the later preps. A single meeting runs inline.

## Output

One table row per meeting: person, date, link to the created page. After the table, one line per
person recapping the briefing highlights (e.g. "Alex: 1 open observation; goals to check: on-call
rotation redesign"). The full briefing lives on the page; don't repeat it in chat.

## Usage

```
1:1 {Person} on {Date} at {Time}
```

One line per meeting; paste several for a batch run. Example prompts:

```
Prepare my 1:1 with Alex tomorrow
```

```
1:1 Sam on Monday at 2:00 PM
1:1 Robin on Monday at 2:30 PM
Skip-level 1:1 - Alex on Tuesday at 10:30 AM
```

```
Prepare today's 1:1 with Sam. Additional context: I want to discuss standardizing dev processes across teams and need their support.
```

## Edge cases

- **No meeting history found**: try the alternative search patterns; if still nothing, create the
  page from the template only and note "Limited meeting history available".
- **Custom context provided** ("I want to discuss X"): add it as a clearly labeled section (e.g.
  "### Custom Topics for Today") after the suggested questions, before the AI prep notes.

