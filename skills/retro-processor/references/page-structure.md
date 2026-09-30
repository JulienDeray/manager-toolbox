# Retro page contract: child page + index row

> The locked shape of the retro's Confluence record. The snippets below are real Confluence
> storage-format markup (from a first run's output, sanitized). Build the child-page body from
> these; the write mechanics live in [wrapup.md](wrapup.md).

## Contents

- Child page: title and section order
- Storage snippets
- Index row (on the Retrospectives index page)
- Formatting rules

## Child page: title and section order

**Title:** `Retro DD-MM-YYYY: <short title>`. The short title names the retro's arc (e.g.
"the quarter we shipped the new pipeline") and is confirmed at the checkpoint. Keep the separator consistent
across every retro page: the index links match on the exact title.

**Parent:** the Retrospectives index page (`retro_index_page` in TEAM_CONTEXT.md). **Space:**
your team's Confluence space.

Section order (all six, always, in this order):

1. **Info panel** (no heading): Date · Attendance · Scope · Cadence, then a Sources line linking
   the transcript doc and noting the board export.
2. `<h2>Summary</h2>`: the few-sentence synthesis (see [synthesis.md](synthesis.md)).
3. `<h2>Main topics discussed</h2>`: one `<h3>` per topic, vote count in the heading where
   known, attributed prose.
4. `<h2>Decisions (aligned directions)</h2>`: a Confluence **decision list**, one item per
   aligned decision.
5. `<h2>Action items</h2>`: the theme table, then a **Parked:** paragraph, then **Small
   one-offs:** as a task list with checkboxes.
6. `<h2>Board snapshot</h2>`: one `<h3>` per board column, the export reproduced verbatim
   (`<ul>` of stickies, bold group names, vote counts kept).

## Storage snippets

Generate fresh `ac:macro-id` / `local-id` / `ac:task-uuid` values (any unique hex/uuid);
everything else is structural.

**Info panel:**

```html
<ac:structured-macro ac:name="info" ac:schema-version="1" ac:macro-id="<uuid4>"><ac:rich-text-body>
<p><strong>Date:</strong> DD-MM-YYYY &middot; <strong>Attendance:</strong> N in the call, M absent
&middot; <strong>Scope:</strong> <period covered> &middot; <strong>Cadence:</strong> every 6 weeks.</p>
<p><strong>Sources:</strong> <a href="<transcript-doc-url>">Meeting notes &amp; transcript</a>
&middot; retro board export (full snapshot at the bottom of this page).</p>
</ac:rich-text-body></ac:structured-macro>
```

**Decision list** (one `decision-item` node per decision; keep the `ac:adf-fallback` `<ul>` in
sync, it is what non-ADF renderers show):

```html
<ac:adf-extension><ac:adf-node type="decision-list"><ac:adf-attribute key="local-id"><hex></ac:adf-attribute>
<ac:adf-node type="decision-item"><ac:adf-attribute key="local-id"><hex></ac:adf-attribute>
<ac:adf-attribute key="state">DECIDED</ac:adf-attribute><ac:adf-content><decision text></ac:adf-content></ac:adf-node>
</ac:adf-node><ac:adf-fallback><ul class="decision-list"><li><decision text></li></ul></ac:adf-fallback></ac:adf-extension>
```

**Theme table** (wide layout; bold theme names, votes in parentheses where known):

```html
<table data-layout="wide"><tbody>
<tr><th><p>#</p></th><th><p>Theme</p></th><th><p>Owner(s)</p></th><th><p>Next step</p></th></tr>
<tr><td><p>1</p></td><td><p><strong><Theme></strong> (N votes)</p></td><td><p><Owner(s)></p></td><td><p><next step></p></td></tr>
</tbody></table>
```

**Parked + one-offs** (task ids are sequential integers, uuids fresh; keep the
`placeholder-inline-tasks` span, it is how Confluence renders inline tasks):

```html
<p><strong>Parked:</strong> <topic> (N votes): <why parked>; revisit at the next retro.</p>
<p><strong>Small one-offs:</strong></p>
<ac:task-list ac:task-list-id="<hex>">
<ac:task><ac:task-id>1</ac:task-id><ac:task-uuid><hex></ac:task-uuid><ac:task-status>incomplete</ac:task-status>
<ac:task-body><span class="placeholder-inline-tasks"><Name>: <commitment>.</span></ac:task-body></ac:task>
</ac:task-list>
```

**Board snapshot:** `<h3>` per column name as exported, then
`<ul><li><p><strong><Group name></strong> (N votes): <stickies, semicolon-joined>.</p></li>...</ul>`.
Ungrouped stickies are plain `<li><p>...</p></li>` items. Reproduce content verbatim: the
snapshot is the record, not a rewrite.

## Index row (on the Retrospectives index page)

The index page holds an intro paragraph and one `Retro index` table with **three columns:
Date | Retro | Key themes**. One row per retro, **newest first** (the new row goes directly
after the header row).

Row shape: the Retro cell is an `ac:link` to the child page **by content-title** (both the
attribute and the link body carry the exact page title); Key themes are `&middot;`-separated:

```html
<tr><td><p>DD-MM-YYYY</p></td>
<td><p><ac:link><ri:page ri:content-title="Retro DD-MM-YYYY: <short title>" /><ac:link-body>Retro DD-MM-YYYY: <short title></ac:link-body></ac:link></p></td>
<td><p><theme 1> &middot; <theme 2> &middot; <theme 3> &middot; <theme 4></p></td></tr>
```

These `ac:link` cells are why the index page must be edited via the REST storage round-trip,
never the MCP HTML one: see [wrapup.md](wrapup.md).

## Formatting rules

- Dates in `date_format` (from TEAM_CONTEXT.md) everywhere.
- **No em dashes** anywhere: commas, colons, parentheses, or `&middot;` separators instead.
- **No links to personal tools** (a private notes app, a personal drive) in this team-facing
  artifact; keep human-readable provenance instead ("raised by Alex in the session").
- Names use the Step-0 roster's canonical spellings (entity-encode accents, or write UTF-8
  directly; both are valid storage).
