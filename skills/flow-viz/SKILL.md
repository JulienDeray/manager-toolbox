---
name: flow-viz
description: |
  Regenerates the team's flow visualization: a self-contained dark HTML artifact ("River + Loom")
  that replays roughly 12 months of ticket history as an animated river of ticket-particles flowing
  through Next, In progress, Review, and Done, where congestion bulges are the bottleneck signal,
  synced with a calendar-time tapestry where thread length equals lead time. Used in team rituals
  (planning, retro) to look at cycle time, lead time, and bottleneck history.
  Use when the user asks to regenerate or refresh the flow viz, the flow visualization, the river, or
  to look at cycle time, lead time, or bottleneck history.
---

# Flow viz: regenerate the River + Loom artifact

One-shot, read-only against Jira; the only outputs are local files under `output/` (gitignore it).
No approval gate needed: nothing is sent or written anywhere shared.

**What the artifact is.** A single self-contained HTML file (data baked in, works offline forever,
open in any browser) with two synced views over the same ticket history:

- **The river**: an animated replay where each ticket is a particle flowing through the stages
  Next, In progress, Review, Done. Congestion bulges (a stage swelling with particles) are the
  bottleneck signal; you watch where the channel narrows over time instead of reading a cumulative
  flow diagram.
- **The loom**: a calendar-time tapestry where each ticket is a horizontal thread; thread length is
  lead time, so eras of long threads are visible at a glance and line up with the river's playhead.

This toolbox does not ship the HTML generator or the extraction script. What it ships is the design:
the data pull, the locked semantics, and the interaction features, so you (or Claude, in a session)
can rebuild the artifact for your own team. The pieces you need to implement are marked below; each is
a small, deterministic script (helper script recommended; see the README caveats).

## Requirements

- **Jira REST read access** for the extraction (issue search plus per-issue changelogs). MCP search
  tools are not a substitute: they can silently truncate a 12-month pull.
- **A shell that can run the scripts you implement** (the extractor and the build/bake step) and
  write to `output/`.
- **A browser** to open the generated HTML.

### Credentials

The extraction needs Jira REST credentials (an email + API token pair) in the repo-root `.env`,
sourced in every Bash call that runs the extraction (shell state does not persist between calls, and
git worktrees do not carry the main checkout's `.env`; resolve the main repo root from the git common
dir when in a worktree). If the variables are missing, stop and ask the operator to add them; never
invent or extract credentials from anywhere else.

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root:

- `jira_project_keys`: the projects to include (tasks, stories, and bugs only; no epics, no subtasks).
- `status_stage_map`: the mapping from your live Jira status names to the canonical stages
  (Backlog, Next, In progress, Review, Done, plus any on-hold statuses).
- `jira_site_url`: for ticket links inside the artifact.

## Steps

1. **Extract** (network, read-only): pull roughly 12 months of ticket history for `jira_project_keys`
   via Jira REST: the issue search plus each issue's status changelog (paged; re-pull truncated
   changelogs). Strictly read-only (search and changelog GETs only). Save the raw JSON to
   `output/`, and report the per-project counts; if a project's count looks implausible (e.g. zero),
   stop and investigate before building.
2. **Build + bake** (compute-only, no network): map statuses to stages via `status_stage_map`,
   compute the clocks (semantics below), and bake the data into the self-contained HTML template.
3. **Unknown statuses**: if the build encounters a status missing from `status_stage_map`, it must
   flag it loudly (never silently bucket it); extend the map and re-run.
4. **Open the artifact** and sanity-check: the ticket count in the header matches step 1's counts;
   playback runs; known history landmarks look right (e.g. a project consolidation shows as
   tributaries converging).

## Semantics (locked; don't quietly change them)

These were argued out once; changing them silently makes every past reading of the artifact
incomparable.

- **Clocks use pass-through imputation** (the standard kanban convention: a skipped stage counts as
  passed through at the transition moment). Lead time = first entry into any flow stage to Done
  (flagged as a fallback when that entry skipped Next); cycle time = first entry into In progress or
  Review to Done; a ticket closed straight from Next gets a 0-day cycle. This guarantees lead >= cycle
  per ticket and one shared percentile population. Tickets that jump backlog-to-Done carry no clocks.
  Reopens: the last Done wins.
- **Blocked** = the union of on-hold status intervals and flagged-field changelog intervals. It is a
  visual state, not a stage.
- **Color = sub-area of origin.** A ticket migrated between projects keeps its origin's color (the
  key changelog tells you where it came from).
- **Assignees appear in tooltips only. Never add per-person aggregates, rankings, or filters.** The
  artifact must stay about flow, not individuals. This is the rule the whole thing hangs on: the
  moment a flow artifact can rank people, it stops being safe to show in a retro.

## Dig-in tooling (all client-side, inside the artifact)

- **Sediment**: particles sink to the channel bed once their stage-age passes a *normative*
  threshold (defaults that worked well: Next 14d, In progress 21d, Review 7d; these are intent, not
  history; document the values in the artifact footer).
- **Filters**: sub-team chips hide-and-recompute (an honest per-team river); the blocked chip is a
  spotlight (dims non-blocked particles, geometry untouched). Still never a per-person filter.
- **Window slice**: two scrubber handles plus presets (4w / 12w / 6m / 1y); counters clamp to the
  slice and relabel.
- **Tile sparklines**: each counter tile carries a full-history daily trend sliced to the window,
  sharing the playhead as a cross-metric cursor (spot WIP-up coinciding with throughput-down); team
  chips recompute the series, the blocked spotlight doesn't; cycle and lead draw P50 plus a faint P85
  with gaps where fewer than 3 samples exist.
- **Actionables drawer**: uncommitted / blocked / in-review / stale lists at the playhead date, click
  to pin, keys linking to Jira. The stale list needs each issue's `updated` field, which is a
  now-snapshot, not history, so it only renders at the data end.

## Usage

```
/flow-viz                           # extract, build, and open a fresh artifact under output/
```

## Notes

- The extraction script is the only network step; keep it strictly read-only.
- If you implement the build as a script, pin its deterministic core with tests (stage mapping,
  clocks, blocked-interval unions, window filtering, baking) against fixtures, so semantics changes
  are loud.
