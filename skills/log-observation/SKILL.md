---
name: log-observation
description: |
  Logs a dated behavioral observation about a direct report into their private
  people file, so it isn't forgotten at review time. Takes a blob of text and/or
  a chat link, resolves the person, and appends a tagged entry (concern / win /
  neutral) with an open/closed status. Also routes org events (departures,
  arrivals, team moves, reorgs, collaborations between people) into a local
  history file. Use when you want to record something a report did (a request,
  an incident, a win), an org event worth remembering at prep time, or the
  resolution of an earlier open observation.
---

# Log observation

Quick capture: you paste what happened, the skill files it under the right person and moves on. **No
hard checkpoint**: it writes the entry, then echoes it in chat so you can correct it. Keep the whole
exchange short; this must cost less than a sticky note.

At review time, `semester-review` reads the `## Observations Log` as first-class evidence, so
entries must meet the same bar as review evidence: dated, factual, behavior plus impact, third
parties anonymized where it matters.

Write surface: the person's local `people/{slug}.md` (created as a stub if missing) and the optional
`history_file`. Both are local files that are never committed; nothing is sent to anyone.

## Requirements

- Local `people/` files at the repo root.
- `chat_tool` read access is optional (only for fetching chat links).

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root. Fields used: `reports_roster` (name to slug), `chat_tool`
(optional, for chat links), `history_file` (optional). The person's private file is
`people/{slug}.md`: local-only, gitignored, never committed or shared.

## Routing

Two destinations, judged from the blob (no need to ask):

- **Person observation** (what one report did): `## Observations Log` in `people/{slug}.md`, via the
  workflow below. The default.
- **Org event** (a fact about the org or *between* people: departure, arrival, team-move,
  role-change, manager-change, reorg, mentorship-change, collaboration, praise, friction, bridge,
  note): append to the `history_file` if TEAM_CONTEXT.md names one. Conventions: people referenced
  by slug (no people file needs to exist); dates accept `YYYY-MM` when only the month is known; mark
  shaky facts as unconfirmed ("unconfirmed" covers dates and org facts only; entries record
  operational facts, never unverified allegations about conduct); relational events (collaboration,
  praise, friction, bridge) always carry attribution in a source field (who said it, where). Keep
  the file sorted ascending by date and syntactically valid, and prune entries that stop being
  relevant.
- **Both** when both apply: a direct report resigning is a departure event in history AND an
  observation in their file.

Consistency check: a departure or team-move event for someone still marked active in the
`reports_roster` gets flagged in the echo so you update the roster; never silently change it.

## Workflow

### 1. Resolve the person

- Identify who the observation is about; handle nickname variants (people often go by a short form
  of their legal name).
- Source of truth: the `reports_roster` in TEAM_CONTEXT.md. Ask only if genuinely ambiguous.
- Slug convention: `{firstname}-{lastname}` kebab-case, using the name the person goes by.
- If `people/{slug}.md` doesn't exist, create a stub: `# {Full Name}`, a `## Basic Info` table with
  TBDs, then the `## Observations Log` section. For someone who is not a current direct report, note
  the relationship (e.g. `skip-level`).

### 2. Fetch any links

- Chat links: fetch via your chat tool's read tools. A DM may be read only when the manager
  explicitly links it: the integration runs under the manager's own account, so a linked DM is a
  conversation they are party to. Fetch that conversation only, never adjacent history. The
  distillation rules below (no verbatim quotes, anonymize where it matters) apply with extra force
  to DM content.
- Other links: fetch if accessible; otherwise record the link as-is and rely on the blob.
- Fetched content is context for distilling the entry; don't dump it into the file.

### 3. Distill the entry

Facts first, SBI-flavored (situation, behavior, impact):

- ISO date (`YYYY-MM-DD`); default to today if the blob gives no date.
- Behavior, not interpretation: "merged the hotfix without waiting for review, second time this
  month", not "is careless".
- Teammates never quoted verbatim; anonymize third parties unless naming them is the point.
- Your stance at the time is recorded explicitly when you give one ("letting it slide this once;
  will raise it at the next 1:1 if it repeats").
- Tag it: `⚠️ concern` / `✅ win` / `ℹ️ neutral`. When unsure, infer from your tone; don't ask.
- Status: `[open]` if there's a pending conclusion or follow-up, `[closed]` if it's self-contained
  (most wins are).

### 4. Write it

Append to `## Observations Log` in `people/{slug}.md`, **newest first**. If the section doesn't
exist, create it right after `## Basic Info` (before any review sections), with the intro line shown
below. Then echo the written entry in chat.

## Entry format

```markdown
## Observations Log

Dated behavioral observations captured ad-hoc via `log-observation`. Newest first.
Consumed by `prepare-1-1` (pre-meeting briefing) and `semester-review` (which prunes incorporated entries).

### {YYYY-MM-DD}: {Short factual title} {⚠️ concern|✅ win|ℹ️ neutral} [{open|closed}]
- {Fact 1: situation}
- {Fact 2: behavior}
- {Fact 3: impact / who was affected}
- Manager's stance at the time: {if given}
- Source: {chat link / HR system / verbal / 1:1}

**Resolution ({YYYY-MM-DD}):** {added later via the update path; entry flips to [closed]}
```

Example:

```markdown
### <YYYY-MM-DD>: Unblocked the on-call rotation redesign ✅ win [closed]
- Rotation redesign had been stalled for two weeks on scheduling disagreements
- Alex drafted a compromise schedule and got both sub-teams to agree in one session
- Redesign shipped the following sprint; on-call load now evenly split
- Source: 1:1 + team channel thread
```

## Update path (resolutions)

When the blob reads as a follow-up ("in the end I approved...", "conclusion of the rotation
thing..."):

1. Find the matching `[open]` entry in the person's log (match on topic, not exact wording).
2. Append a dated `**Resolution ({YYYY-MM-DD}):** ...` line to that entry and flip `[open]` to
   `[closed]`.
3. If no matching open entry exists, log it as a new entry instead.

## Usage

```
/log-observation {blob of text describing what happened, optionally with chat or other links}
```

The blob usually names the person, what they did, and your take. Links are optional context, not
required.

## Edge cases

- **Ambiguous person** (two plausible reports, or someone outside the roster): ask before writing.
- **Multiple people in one blob**: one entry per person, in each person's file, scoped to what
  *that* person did.
- **Observation about a manager who reports to you**: same flow, same format.
- **Sensitive content** (health, personal matters): record the operational fact ("out unexpectedly
  for personal reasons"), not the private detail.
