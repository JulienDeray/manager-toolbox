# Team Context (template)

Copy this file to `TEAM_CONTEXT.md` at your project root (the directory you run Claude Code from)
and fill it in; add the filled copy to your repo's `.gitignore`. Every skill in this toolbox reads `TEAM_CONTEXT.md` as its Step 0 and takes all
identifiers from it. Fill in your own values; never hardcode an ID inside a skill. If a skill needs
a field that is missing here, it should stop and say so rather than guessing. Skills you don't use
leave their fields blank.

## Jira

| Field | Value | Notes |
|-------|-------|-------|
| `jira_site_url` | `https://<your-site>.atlassian.net` | Base URL for browse links. |
| `jira_project_keys` | `PROJ, OPS` | The team's project keys; the first is the primary delivery project. |
| `backlog_project` | `PROJ` | The project whose Backlog holds ideas (the resurfacing pool). |
| `routing_rules` | e.g. "work likely active > 1 week -> PROJ; short-lived ops / on-call -> OPS" | Which project takes which kind of work. |
| `definition_of_ready` | e.g. "clear summary + description, an assignee, one work-type label, dependencies identified" | Required before an issue advances past backlog. |
| `status_stage_map` | e.g. `Backlog->Backlog, Ready->Next, In Progress->In progress, In Review->Review, Done->Done, On Hold->on-hold` | Live status names to canonical flow stages. |
| `resurfacing_exclusions` | e.g. `labels not in ("owned-elsewhere")` | Optional: items the save-or-die review must skip. |
| `epic_board_id` | board ID | The epic planning board (planning-processor / planning-agenda scope). |
| `delivery_board_id` | board ID | The task/delivery board (touched lightly; excluded from the WIP count). |

## Label ontology

| Field | Value | Notes |
|-------|-------|-------|
| `team_ontology_page` | a Confluence page ID, or the taxonomy inlined here (optional for intake) | The label axes + heuristics, e.g. work-type `wt::` (exactly one per issue, priority-ordered heuristics), area `area::`, service `svc::`, weekly focus `prio::now`, migrations `migration::<slug>`, strategy `axis::` (required at the commitment line: the move from Backlog into committed work, e.g. Next). Include carve-outs and retired labels. |

No ontology yet? Start with this two-axis one inline and grow it later:

- `wt::project`: new capability. `wt::ops`: maintain, upgrade, automate. Exactly one `wt::` label per issue.
- `area::<sub-team>`: optional, one per issue when the team has sub-areas.

## General

| Field | Value | Notes |
|-------|-------|-------|
| `date_format` | e.g. `DD-MM-YYYY` | Used consistently in comments, register cells, and messages. |
| `team_timezone` | e.g. `Europe/Zurich` | For scheduled-task slots. |
| `planning_slot` | e.g. `Mon 10:00` | The weekly planning/grooming slot (planning-agenda posts ~40 min before it). |
| `standup_comment_prefix` | e.g. `[STANDUP DD-MM-YYYY]` | Prefix for stand-up context comments on Jira issues. |
| `planning_comment_prefix` | e.g. `[PLANNING DD-MM-YYYY]` | Prefix for weekly planning comments on focus items. |
| `transcript_source` | e.g. "Google Meet + Gemini notes docs in Drive; stand-up titled `<team> stand up`, planning titled `<team> Week Planning`" | Where meeting transcripts land and their title patterns, for the auto-fetch steps. |
| `hr_availability_source` | describe how to query who's out for a date range: an HR-system API, an export, or a shared leave calendar | Optional: the who's-out source for planning-agenda; omit to skip availability. |

## Slack

| Field | Value | Notes |
|-------|-------|-------|
| `team_channel` | `C0XXXXXXX` | The team channel (digests, resurfacing posts, outcome posts, wrap-ups, agendas). |
| `leadership_channel` | `C0XXXXXXX` | The engineering-leadership channel (migration digest). |
| `manager_slack_id` | `U0XXXXXXX` | Tagged as the cancellation-confirming manager; also how "the manager" is resolved. |
| `stakeholder_channel` | `C0XXXXXXX` | Optional: standing target for stakeholder-facing stand-up messages (`--external`); ask per run if absent. |

## People & teams

| Field | Value | Notes |
|-------|-------|-------|
| `team_roster` | table: name, Slack ID, Jira account ID, sub-area | For attribution, tagging, and assignee resolution. |
| `engineering_teams_roster` | table: team, Jira project key, project type (company-managed / team-managed), team Slack channel | Drives migration expected-teams proposals, association mechanics, and nudge routing. |

## Confluence

| Field | Value | Notes |
|-------|-------|-------|
| `migrations_register_page` | page ID | The cross-team migrations register. |
| `migration_detail_template_page` | page ID | The migration detail-page template. |
| `team_agreement_page` | page ID | Ways-of-working: board model, WIP limits, ritual rules (stand-up format, planning cadence). |
| `direction_page` | page ID | Optional: the team's current direction statement + active strategy-label subset; feeds the agenda's direction banner and the direction-linkage checks. |
| `wip_metrics_page` | page ID | The weekly WIP & flow-metrics history table (planning-processor appends; planning-agenda reads). |
| `stepback_page` | page ID | The Quarterly Step-Back page: session config (`Next step-back` date, board link, instructions) + History table. |
| `retro_index_page` | page ID | The Retrospectives index page (intro + index table); retro child pages nest under it. |

## Transcript-garble glossary

A table mapping recurring auto-transcription garbles to their decoded meanings, read **and
maintained** by every skill that parses a transcript (the processors and wrap-1-1). Each such
skill decodes against the table before attributing anything, and proposes a new row when a run
decodes a recurring garble: every row is confirmed by the operator at that skill's plan or
checkpoint, never silently added, and zero new garbles is a valid outcome.

| Garbled as | Decoding | Added |
|------------|----------|-------|
| "Cooper Netties" | **Kubernetes** (example row; replace with your own) | DD-MM-YYYY |

## Conventions

Generic rules every skill follows; skills point here instead of repeating them and keep only their
own message shapes, section lists, and caps.

### Slack message formatting

The MCP message tools parse standard markdown and convert it on send.

- Bold is `**double asterisks**`, never `*single*` (a literal `*single*` posts as italic).
- Sub-topic labels are bold with a colon: `**Observability:** scrape lag fixed`.
- Lists are `-` bullets; number them only where the numbering is load-bearing.
- No blank line between bullets within a list: a blank line between `-` items makes Slack parse
  each as its own single-item list, stacking list margins, and the message sprawls over multiple
  screens. Blank lines separate sections only.
- **Every cited Jira key must be a markdown link**, `[PROJ-101](<jira_site_url>/browse/PROJ-101)`,
  never bare text, so readers can jump straight to the ticket.
- No em dashes anywhere in a message; use colons, commas, parentheses, or a `·` separator.
- **Pre-send self-check** before any draft or send call: no blank line between consecutive bullets;
  titles and labels are `**...**`; no em dash; every ticket key is a link.
- Never prototype by posting.
- **One draft per channel:** Slack allows only one attached draft per channel. A second draft call
  to the same channel silently overwrites the first with no error, so build one combined draft
  (with a clearly dated section per day when a run covers several days).

### Transcript hygiene

For every skill that parses a transcript or meeting notes, against the glossary above:

- Decode the transcript against the transcript-garble glossary before attributing anything
  (auto-transcription mangles names and product terms; the glossary carries the canonical
  spellings).
- Collect new garble candidates: a recurring mis-transcribed name or term whose meaning the context
  makes clear, each with its source quote.
- Propose them as glossary rows, one confirm each, never silently added.
- Zero new garbles is a valid outcome.
- Append confirmed rows to the glossary table.

## People management

Used by the people skills (prepare-1-1, wrap-1-1, log-observation, semester-review, create-pdp,
intake-for-me, pull). Per-person data lives in local files: `people/<slug>.md` (the manager's
private knowledge base per report), `people/reviews/<slug>-<cycle>/` (review evidence and drafts),
and `notes/` (your personal knowledge base). These directories are gitignored and must never be
committed, shared, or pasted into shared surfaces.

| Field | Value | Notes |
|-------|-------|-------|
| `reports_roster` | table: name, slug, role (IC / manager), relationship (direct / skip-level), active | Drives the people skills; the slug names the person's `people/<slug>.md` file. |
| `notes_tool` | e.g. "Obsidian", "a Confluence personal space", "a wiki" | Where 1:1 prep and meeting pages live. |
| `meeting_title_convention` | e.g. `1:1 {FirstName} - {YYYY-MM-DD}` (prepare-1-1's default) | How the skills find and create 1:1 pages. |
| `opener_questions` | a short list | Rotating 1:1 opener questions for prep pages. |
| `chat_tool` | e.g. Slack | Where summary DMs are drafted (drafted only; you send). |
| `source_forge` | e.g. "GitLab; usernames per report in `reports_roster`" | Evidence source for reviews. |
| `review_cycle` | e.g. "semesters (H1 / H2), company review form questions inlined here" | Cadence plus the form the review must answer. |
| `history_file` | optional, e.g. `notes/org-history.md` | Org-event log (reorgs, role changes) that observations can route to. |
| `pdp_storage` | e.g. "Google Docs, one doc per person per cycle" | Where development plans live and how they're shared. |
| `team_goals_file` | path or page | Company / engineering / team goals that PDP goals must align to. |
| `task_system` | e.g. "your task manager" | Where dated personal tasks from intake-for-me land. |
| `team_intake_pipeline` | optional | The hand-off target for team-bound ideas (this toolbox's `intake` skill, if you use it). |
| `style_rules` | e.g. "terse notes; no em dashes; bold names" | Personal writing-style rules the drafting skills must respect. |
