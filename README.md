# manager-toolbox

A set of [Claude Code](https://claude.com/claude-code) skills for running an engineering team: team rituals, flow health, cross-team programs, and people management.

These are not prompt snippets. Each skill is a versioned, reviewable procedure that an AI agent executes with real access to Jira, Confluence, and Slack: it reads the same sources you would, follows the same edge-case rules you would, and hands you the judgement calls. They were built and hardened over months of daily use on a real team, then sanitized for publication. The config is externalized, the war stories stayed in.

## Start here

1. **Install one skill.** Create a private repo you will run Claude Code from, and copy `skills/intake/` into its `.claude/skills/`.
2. **Fill only what intake reads.** Copy `TEAM_CONTEXT.template.md` to `TEAM_CONTEXT.md` in that repo and fill the fields intake's Step 0 lists: `jira_project_keys`, `backlog_project`, `routing_rules`, `definition_of_ready`, `team_channel`, `team_roster`, `jira_site_url`, plus `team_ontology_page` if you have one (the template carries a two-axis starter). Leave everything else blank.
3. **Paste a Slack message and say "file this".** Intake classifies it and proposes where it belongs; nothing is written until you approve.

Then rehearse the scheduled skills without writing anything: `/standup-processor dry-run` and `/backlog-resurfacing dry-run`.

## The three invocation modes

How a skill gets invoked is a design decision, not an accident:

1. **Scheduled, no invocation at all.** Wired to cron. The skill fires, does its work, and either posts its output directly (informational digests and briefings) or leaves a draft for a human to send. Examples: `planning-agenda` (40 minutes before planning), `backlog-resurfacing` (Wednesday), the `migration-tracker` daily nudge check.
2. **Manually invoked rituals.** Fired by name with meeting notes or a transcript as input, because they need judgement and write a lot: `planning-processor`, `retro-processor`, `stepback-processor`, `wrap-1-1`, `semester-review`.
3. **Auto-triggered by wording.** The skill's description is written so the agent picks it up from how you phrase the request: paste a message and say "file this" and `intake` fires; "fix the labels on PROJ-101" fires `ticket-hygiene`.

## Design rules

Four rules made this sustainable. They are enforced inside the skills, not just documented here:

- **The AI drafts, a human presses send.** Every skill declares, per write class, whether it fires, waits for approval, or only ever produces a draft. On shared team surfaces only informational output and factual bookkeeping fire: scheduled briefings, prefixed ticket comments, clear-case ticket creation, due-date stamps, register rows, append-only hygiene. Two scheduled skills post straight to the team channel: `planning-agenda` (a briefing) and `backlog-resurfacing` (the weekly vote post). Inside a firing class, anything ambiguous is reported instead of written. There are no trust levels or modes to switch at runtime: changing a declaration is an edit to the skill, reviewed like any other change. Nothing that evaluates a person's work or cancels work ever executes unattended, and wrap-ups and anything addressed to a person leave as drafts a human sends.
- **Skills invoke each other instead of duplicating procedures.** `stepback-processor` calls `planning-processor`, which calls `intake` and `ticket-hygiene`. Each skill keeps a single job, and each invoked skill brings its own gate with it.
- **Config lives in one file.** Every identifier (channels, project keys, page names, cadences, roster) comes from `TEAM_CONTEXT.md`, read as Step 0 of every skill. Nothing is hardcoded, which is also what made this repo publishable.
- **Carry the finding, not the rows.** Skills that touch people data keep it local (`people/` is gitignored and never shared), never aggregate flow metrics per person, and put only conclusions into shared surfaces.

## Skill catalog

Four categories, one management pain each.

### Capture: "we should fix that" is where ideas go to die

| Skill | Mode | What it does |
|---|---|---|
| [intake](skills/intake/SKILL.md) | auto-trigger | Single front door for unstructured input: classifies any pasted message, transcript, or idea against your label ontology and proposes the right home (ticket, epic, backlog idea, migration). |
| [intake-for-me](skills/intake-for-me/SKILL.md) | auto-trigger | Captures your own ideas, questions, and observations into a personal knowledge base, including markers dropped inline in meeting notes. |
| [pull](skills/pull/SKILL.md) | manual | Surfaces 3-5 open ideas with a why-now rationale when you have slack time, and gardens the notes. |

### Rituals: ceremonies that change the system, not just the mood

| Skill | Mode | What it does |
|---|---|---|
| [standup-processor](skills/standup-processor/SKILL.md) | scheduled | Turns the daily standup transcript into Jira comments and new tickets, plus a Slack wrap-up. Factual writes fire, anything ambiguous is reported instead of written, and the wrap-up is a draft a human sends. Rehearse with `dry-run` before scheduling. |
| [planning-agenda](skills/planning-agenda/SKILL.md) | scheduled | Pre-planning briefing before the weekly session: last week's focus review, who's out, backlog pull-candidates. |
| [planning-processor](skills/planning-processor/SKILL.md) | manual ritual | After weekly planning: sets the week's focus labels, posts the wrap-up with WIP and flow metrics, files raised ideas, ticket hygiene. |
| [retro-processor](skills/retro-processor/SKILL.md) | manual ritual | Retro wrap-up: board export plus transcript into a durable record, actions consolidated into 3-4 owned themes, Slack draft. |
| [stepback-processor](skills/stepback-processor/SKILL.md) | manual ritual | Quarterly step-back wrap-up: alignment verdict on long-term direction, history entry, next-date bump, then invokes planning-processor. |

### Truth: the board stays honest

| Skill | Mode | What it does |
|---|---|---|
| [backlog-resurfacing](skills/backlog-resurfacing/SKILL.md) | scheduled | Weekly save-or-die review: the 5 oldest backlog items go to the channel; a reaction saves, silence proposes cancellation, and cancellations are never executed unattended. |
| [ticket-hygiene](skills/ticket-hygiene/SKILL.md) | auto-trigger | Preview-then-approve description and label fixes on discussed tickets; the hygiene hand-off other skills invoke; standup-processor's hand-off applies only its append-only classes without the gate. |
| [flow-viz](skills/flow-viz/SKILL.md) | manual | Animated replay of ~12 months of ticket flow where congestion reveals bottlenecks. Never shows per-person aggregates, by design. Ships the design only: the extractor and generator are not included. |
| [migration-tracker](skills/migration-tracker/SKILL.md) | scheduled + manual | Tracks org-wide migrations your team leads: onboarding, per-team rollups, drafted nudges to lagging teams, thread replies folded back into the register. |

### People: judgment on evidence, delivery by humans

| Skill | Mode | What it does |
|---|---|---|
| [prepare-1-1](skills/prepare-1-1/SKILL.md) | manual | Builds the 1:1 prep page from meeting history, with a private pre-meeting briefing above a divider; only what is below the divider is shared. |
| [wrap-1-1](skills/wrap-1-1/SKILL.md) | manual ritual | After the 1:1: completes your terse notes from the transcript, drafts the summary DM (you send it), logs observations, closes the loop. Pairs with `prepare-1-1`, which creates the page it reads. |
| [log-observation](skills/log-observation/SKILL.md) | auto-trigger | Logs a dated observation (concern, win, or neutral; open or closed) into a person's file so reviews aren't memory-based. |
| [semester-review](skills/semester-review/SKILL.md) | manual ritual | Collects and triangulates review evidence (1:1 history, forge activity, peer and upward feedback) and helps you tighten your own draft of the manager feedback (it never drafts from scratch), with explicit checkpoints. |
| [create-pdp](skills/create-pdp/SKILL.md) | manual | Two-stage personal development plan: skeleton right after the review, intermediate goals proposed a few weeks later. |

## Setup

1. **Prerequisites**: Claude Code with MCP connectors for your stack. Assumed: Jira + Confluence (Atlassian MCP) and Slack. Also needed per skill:
   - the people skills need MCP access to wherever your 1:1 pages live (`notes_tool`);
   - `flow-viz` and `retro-processor` need an Atlassian API token in a repo-root `.env` (gitignored here);
   - the gated skills use Claude Code's plan mode as their approval step;
   - optional: an HR system for availability, a transcript source (e.g. Google Meet) for the ritual processors.
2. **Install**: copy the `skills/` directories you want into `.claude/skills/` of the repo that holds your `TEAM_CONTEXT.md` (or into `~/.claude/skills/` to make them personal). Either way, run Claude Code from the directory that holds `TEAM_CONTEXT.md`: every skill reads it from there.
3. **Configure**: copy `TEAM_CONTEXT.template.md` to `TEAM_CONTEXT.md` in that directory and fill in the fields for the skills you use. Every skill reads it as Step 0.
4. **Keep private data out of git**: add `TEAM_CONTEXT.md`, `people/` and `notes/` to your repo's `.gitignore` (copy this repo's). The people-management skills keep per-person files under `people/`, and that directory must stay private: never commit it, never paste it into shared surfaces.
5. **Schedules**: wire the scheduled skills to Claude Code scheduled tasks (or cron) at the cadences suggested in each SKILL.md. Start with `dry-run` wherever a skill offers one, then schedule the ones you trust. Run it again after changing the roster, ontology, or glossary.

## Terms

- **Operator**: the person running the Claude Code session.
- **Manager**: the person who sends drafts and approves cancellations and checkpoints (resolved via `manager_slack_id`). Often the same person as the operator.
- **Gate / plan-approve**: the skill builds its full plan read-only, shows it, and writes nothing until the operator approves.
- **Fire-without-gate**: a skill (or a write class within one) that writes without an approval step. Each skill declares which of its writes fire.
- **Checkpoint**: a mid-run stop where the skill waits for the operator's confirmation before continuing.
- **Draft**: a real Slack draft in the target channel; the skill never sends it, a human does.
- **Dry-run**: an argument that composes everything the run would write and prints it, writing nothing.
- **"Not written" list**: the part of a run report listing what a firing skill deliberately did not write, with a one-line reason each.

## Honest caveats

- Helper scripts are not bundled yet. Where a skill benefits from one (bulk Jira reads, byte-safe Confluence section edits), the SKILL.md says so and describes the step; without them the skills fall back to MCP search, which can silently truncate large result sets; each skill says where that matters. `flow-viz` ships the design only: you build its extractor and generator.
- This repo is a curated snapshot of a living internal system, not the live copy: the internal versions keep evolving daily, and the public set is re-synced through a sanitization pass periodically. Expect the ideas to be current and the fine print to lag by a little.
- These encode one team's opinions: WIP limits matter, backlogs must earn their size, feedback is drafted by AI but owned and sent by humans. Disagree freely; the skills are markdown, edit them.
- Some runs are context-heavy: transcript processing and reads of 100+ issues. Use a large-context model and expect long runs.
- The transcript-based skills assume meetings are recorded openly and in line with local consent rules; a recorded 1:1 should never be a surprise to the other person.
- An agent with write access to your Jira and Slack deserves the same care as a new hire with admin rights. Read the gates in each skill before you loosen them.
- Issues are welcome. PRs may land slowly, because this repo is re-synced from an internal copy.

## License

MIT. See [LICENSE](LICENSE).
