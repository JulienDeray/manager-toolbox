# Retro wrap-up: write mechanics

> The execute half of the `retro-processor` skill: create the child page, splice the index row,
> land the glossary rows, create the Slack draft, verify everything. Page contract:
> [page-structure.md](page-structure.md). Everything here runs **after** plan approval; the
> gather/synthesize half lives in SKILL.md and [synthesis.md](synthesis.md).

## Contents

- Why two write paths (MCP vs REST)
- W1: Create the child page (MCP)
- W2: Read the index page (REST storage)
- W3: Splice the index row
- W4: Update the index page (REST PUT)
- G: Glossary maintenance
- S: Slack draft
- W-verify (mandatory)
- Notes & edge cases

## Why two write paths (MCP vs REST)

- **New pages go through the MCP create-page tool**: a fresh body has zero retype-drift risk.
- **Edits to existing pages go through the Confluence REST v2 storage round-trip**, not the MCP
  HTML one, because both target pages defeat the HTML path: the index page's Retro cells are
  `ac:link` elements, and the MCP HTML conversion **drops `ac:link-body`**, so writing the
  round-tripped body back would turn every existing retro link into a bare URL. A large,
  macro-heavy config page (where the glossary lives, if you keep it on a wiki page rather than
  in TEAM_CONTEXT.md) has the same problem: a full-body retype risks silent drift on entities
  and macro ids.

Credentials: your Atlassian email + API token (e.g. from a repo-root `.env`; the same token
works for Confluence and Jira on an Atlassian cloud site). REST here is only `GET`/`PUT` on
`/wiki/api/v2/pages/{id}`.

> **Glossary location note.** In this public toolbox the transcript-garble glossary lives in
> `TEAM_CONTEXT.md`, so the G step below is a plain local file edit (the filled file is
> gitignored: never commit it), and the REST machinery applies only to the index page. The wiki-page variant is kept documented for teams
> that store the glossary on Confluence.

## W1: Create the child page (MCP)

Create the page: your team space, parent = the Retrospectives index page, title and
storage-format body per [page-structure.md](page-structure.md). Capture the new page id from
the response: the report links it and W-verify re-reads it. If the MCP tool mangles the
decision-list / task-list macros (the verify-read shows them missing), fall back to REST v2
`POST /wiki/api/v2/pages` with the same approved body: the same write, different transport.

## W2: Read the index page (REST storage)

Reuse the plan-mode read if still fresh; otherwise:

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" \
  "https://<your-site>.atlassian.net/wiki/api/v2/pages/<INDEX_ID>?body-format=storage" > index.json
jq -r '.body.storage.value' index.json > index.html
jq -r '.version.number' index.json
```

(Work in the session scratchpad; always give `jq` an input file: a bare `jq` in a compound
command blocks forever on stdin.)

## W3: Splice the index row

Insert the new row directly after the header row of the index table (newest first), leaving
every other byte of the body untouched. This is a pure string manipulation on the storage body;
(helper script recommended; see the README caveats). If the
page has more than one table, anchor the splice to the `Retro index` heading rather than the
first `<tbody>`. The Retro and Key-themes cells are pre-built HTML per
[page-structure.md](page-structure.md).

The `ri:content-title` must match the just-created child page's title **byte-for-byte** (entity
encoding included) or the link resolves nowhere.

## W4: Update the index page (REST PUT)

```bash
jq -n --arg title "$(jq -r '.title' index.json)" --rawfile body index-new.html \
  --argjson v "$(( $(jq -r '.version.number' index.json) + 1 ))" \
  '{id: "<INDEX_ID>", status: "current", title: $title,
    body: {representation: "storage", value: $body},
    version: {number: $v, message: "Retro <DD-MM-YYYY>: index row"}}' > put.json
curl -s -X PUT -H 'Content-Type: application/json' -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" \
  -d @put.json "https://<your-site>.atlassian.net/wiki/api/v2/pages/<INDEX_ID>"
```

If the PUT is denied (a permission classifier may block raw-curl writes): do **not** fall back
to the MCP update here; the `ac:link` cells make the HTML round-trip destructive. Surface the
pending row to the operator (the spliced body is sitting in the scratchpad) and stop this step.

## G: Glossary maintenance

Only decodings **confirmed at the checkpoint** are written. **Zero confirmed new garbles is a
clean no-op: report it as such and skip this step.**

With the glossary in `TEAM_CONTEXT.md` (the toolbox default): append the confirmed rows to the
glossary table in the file, newest first, each row carrying the garbled string, the decoded
meaning with a one-line context and the retro date, and the date added.

With the glossary on a Confluence page instead, mind these gotchas:

1. REST GET the page in storage format (as in W2), locate the unique glossary heading anchor,
   find the next `<tbody>` and the end of its header row, and insert the new rows there,
   newest first. **Assert before writing:** the anchor occurs exactly once in the body, and the
   output differs from the input only by the inserted rows.
2. REST PUT with `version.number = previous + 1`.
3. This edit must be an **explicitly-approved row in the plan, never "optional"**: a permission
   classifier may deny a raw-curl PUT that the plan gated as optional.
4. **If the PUT is denied:** fall back to the MCP full-body update **only after checking the
   storage body contains no `ac:link`** (the HTML conversion drops `ac:link-body`, so an
   `ac:link` anywhere on the page makes the fallback destructive). If the check fails or can't
   be done, surface the pending rows to the operator; never retry blindly.
5. **Verify diffs by content, not byte equality**, on pages carrying status macros: Confluence
   reorders macro parameters on save.
6. A REST storage PUT strips `<span data-type="status">` to a plain span; if a status cell is
   ever needed, hand-build the `ac:structured-macro ac:name="status"` form with a fresh
   macro-id instead.

## S: Slack draft

Create the wrap-up as a draft attached to the team channel via the Slack draft tool,
automatically once the plan is approved, no extra confirm step; the manager reviews and sends by
hand. Never call the send tool.

Message shape (in order):

1. Thanks + one-line framing ("retro wrapped up").
2. One or two highlight lines (the period's wins, from the Summary).
3. `**Decided:**` lead + a `-` list of the aligned decisions, in plain language.
4. `**Follow-up themes:**` lead + a `-` list of the 3-4 themes with owners, vote-leader first.
5. A labelled link to the retro page: `[Full retro notes](<child page URL>)`, never a bare URL.
6. A closing "follow-ups coming" line.

Format contract (**list shape**): blank lines between the parts above, but the `-` items inside
a `**Decided:**` / `**Follow-up themes:**` list stay consecutive lines, never blank-separated (a
blank line between `-` items makes Slack split the list and double the vertical gap).
Follow the Slack message formatting conventions in `TEAM_CONTEXT.md` (bold, links, no em
dashes). Additionally for this message: **no internal jargon** (no label syntax, no board column
names; write for the team, not the board); `<@SLACK_ID>` mentions and emoji are fine.

Edge cases: Slack allows one attached draft per channel; if one already exists, report it and
hand over the text instead of failing. If the draft tool is unavailable, the text in chat is the
deliverable; never call the send tool for it.

## W-verify (mandatory)

Re-read everything touched and confirm before claiming success:

1. **Child page** renders with all six sections (decision list and task list intact: see the W1
   fallback), and nests under the Retrospectives index (its `parentId`).
2. **Index page:** the new row is the first data row; its `ac:link` resolves (title matches the
   child byte-for-byte); every byte outside the inserted row unchanged; version +1.
3. **Glossary:** the confirmed rows present, everything outside the edit unchanged (skip if the
   step was a no-op).
4. **Slack:** the draft exists in the channel (or the text was handed over in chat).

Report the reconciled result plainly: links, what landed, what was skipped and why.

## Notes & edge cases

- **Version conflict** (someone edited a page between GET and PUT): re-GET, re-splice on the
  fresh body, retry the PUT once. Never hand-merge HTML.
- **Re-run protection is upstream:** the plan-mode idempotency check (a child page or index row
  already dated with this retro) prevents duplicates; if a verify-read shows a duplicate anyway,
  undo by hand, don't script a deletion.
- **A hand-edited index page is fine**: the scoped splice leaves every other byte alone.
- **No other writes.** No Jira, no board tool; handover docs only via the explicit-ask flow in
  SKILL.md.
