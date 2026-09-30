# Jira routing and the outcome post: detailed procedure

> Capabilities 2 and 4 of the `intake` skill: filing an approved proposal into Jira (the three issue
> shapes, the description, the initial status, the pre-write dedup, execute + verify) and drafting the
> outcome post to the team channel. Examples use the invented project key `PROJ`.

## Capability 2: Route to Jira

### The three issue shapes

Without a `team_ontology_page` (see Step 0 in [SKILL.md](../SKILL.md)), the shapes below carry no
labels: none are proposed or set.

**(a) Ticket (Task or Bug)**: the default for a concrete unit of work. Project per `routing_rules`;
Bug only if the input describes a defect; exactly one `wt::` plus an `area::`; assignee only if the
input names a clear owner (otherwise leave unassigned and flag it); a written description (see below);
proposed initial status.

**(b) Epic**: a large multi-ticket effort. Usually the delivery project; the project-work `wt::` plus
an `area::`; theme, rough breakdown, and provenance in the description. If the input is really an
org-wide change other teams must adopt, it is a migration, not a plain epic: route via Capability 3,
which creates the tracking epic itself.

**(c) Idea**: undecided, filed into the Backlog of `backlog_project`, always as a Task (never an Epic;
sizing happens at promotion, if ever). Labels as usual plus at most one candidate `axis::` label where
one clearly fits (cheap tagging at entry; the labelling obligation sits at the commitment line). No
assignee unless the source names one. Status Backlog, always; never propose advancing an idea. No due
date: on Backlog issues the due date is the resurfacing clock (see the `backlog-resurfacing` skill)
and it starts empty.

### The description (every created issue gets one)

- What / why: the distilled problem statement plus the trigger.
- Acceptance or scope: a line or two if the input supports it; otherwise note it is to be refined in
  grooming.
- Originator attribution in prose, e.g. `Raised by Robin in the weekly sync`.
- A provenance footer, e.g. `_Source: <permalink> · filed by <user> on <date>_`.

### Initial status (+ incident flag)

| Proposed lane | When | Gate |
|---------------|------|------|
| backlog (default) | new, unrefined | none |
| ready / next | clearly ready, or incident-shaped (visibility) | clear summary + description + work-type label; flag if thin |
| in progress | someone is already actively working it | your `definition_of_ready` (assignee, description, labels); surface any gap rather than advancing silently |

Don't hardcode status names: read the live transitions at execution time and map the intent to
whatever the workflow calls it. If no matching transition exists, report it and leave the issue in
backlog rather than forcing it.

### Pre-write dedup (read-only, best-effort)

Search for an existing issue covering the same thing, by summary and distinctive key terms, not by the
raw provenance URL (Jira tokenises URLs unpredictably, so a `description ~ "<url>"` clause misses real
duplicates). Example JQL:

```
project in (<your projects>) AND statusCategory != Done AND summary ~ "<key terms>" ORDER BY updated DESC
```

Run it through a paginated REST search (see Requirements in [SKILL.md](../SKILL.md)): a broad summary match easily exceeds an MCP
row cap, and here truncation cuts against the whole point of the step: the duplicate you never see is
the one you re-file. If likely matches come back, show them in the plan and ask: file as new, skip, or
comment/link on the existing issue instead. Frame it honestly as "possible duplicates, confirm"; it is
not exhaustive, and the approval gate is the real backstop against a double-file.

### Execute + verify

1. Create the issue with the approved payload; capture the key. New issues land in the workflow's
   initial status (do not set status in the create payload).
2. If labels or assignee were not accepted at create, edit to set them, sequentially, never in
   parallel (shuffled-response caveat).
3. If the approved status is past backlog, read the live transitions and apply the matching one.
4. Verify-read the new key: issue type, the single `wt::` plus `area::`, assignee, status, and the
   description with its provenance footer. Claim success only once the verify-read matches the plan.

Edge rules:

- Exactly one work-type label, never two.
- No weekly-focus label by default.
- If your ops project has a formal change-management issue type that outsiders file, intake never
  creates it; work your team needs becomes a regular ticket and the outcome post notifies the
  sub-team.
- Read-only over other teams: if the right home is another team's board, say so and stop (or park it
  as a request to that team); never create there.
- Ambiguous project: ask. A one-line question is cheaper than a mis-filed ticket.

## Capability 4: Announce the outcome

After a successful, verified write, draft a short outcome post to `team_channel` (as a Slack draft). The default varies
by destination, so the channel stays a meaningful feed rather than a firehose:

| Destination | Default |
|-------------|---------|
| Jira ticket / epic | Draft by default: the team should see new committed work landing. |
| Incident-shaped | Always draft, loud (see below). |
| Backlog idea | No post by default: undecided, low signal; the weekly resurfacing review is its visibility surface. Offer one only if asked. |
| Migration | No post from intake: migration-tracker owns its own announcement; don't double-post. |

Draft shapes (one tight line: what + where + link + source):

- Ticket: `:inbox_tray: Filed PROJ-101 (<project>, <work-type>): <summary>. Source: <provenance>. <browse link>`
- Epic: `:inbox_tray: New epic PROJ-102: <theme>. Source: <provenance>. <browse link>`
- Idea: `:inbox_tray: Filed an idea into the Backlog: PROJ-103: <title>. It'll come up in the weekly resurfacing review. Source: <provenance>. <browse link>`
- Incident-shaped, lead with the flag: `:rotating_light: Incident-shaped: PROJ-104 filed (<expedite work-type>, status <lane>). Likely needs on-call NOW; this ticket is not a substitute for paging. <browse link>`

Rules:

- Tag a person (via their Slack ID from `team_roster`) only when there is a clear owner who should
  see it; otherwise no ping.
- Always exact, never compressed: Jira keys in full, links complete and clickable, `<@SLACK_ID>` and
  `<#channel>` refs verbatim, the provenance string intact. Trim words, not facts.
- One post per intake run, summarising the destination (or the small fan-out as one message with a
  line each).
- A human sends. Create the post as a Slack draft in `team_channel` via the draft tool and say where
  it sits; the manager sends it from Slack. If the draft tool is unavailable, present channel + text +
  any mention in chat: that text is the deliverable. Never call the send tool.
- No outcome post without a write: if every proposed write was declined, there is nothing to announce.
