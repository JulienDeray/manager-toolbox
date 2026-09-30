# Jira description reads: traps and recovery

> Reference for the `ticket-hygiene` skill. H1, H2, H5 and the Auto mode all depend on reading a
> description correctly; this file holds the two read traps and the recovery path for a clobbered
> description.

## Reading a Jira description safely (two traps, both destructive)

Two independent read mistakes both render a full description as "empty", and H2 branches straight
from "empty" to "write one", so a bad read can silently clobber a 2,000-character description with
a generated one. The write succeeds, so nothing fails loudly.

1. **`expand=renderedFields` renders only the fields named in `fields`.** With a fields list that
   accidentally omits `description`, `renderedFields.description` comes back null for a ticket
   with a full description. Whenever you intend to read a description, name `description` in
   `fields`.
2. **`fields.description` is an ADF object, not a string.** Its length is 3 (the `type`, `version`,
   `content` keys) and every substring check against it fails, so a naive emptiness check "passes"
   and a naive verification check "confirms". Flatten the tree first: walk it collecting
   `text`-node values, then judge the flattened string. (Some MCP get-issue tools return markdown
   directly; a raw REST read does not.)

Read through a helper that always requests `description` and returns it pre-flattened, plus an
explicit `is_empty` boolean (helper script recommended; see the README caveats).

## Recovering a clobbered description

Kept here because this skill is the one that can cause it: the prior text survives in the issue
changelog. Read the changelog's description history newest-first; the top entry's `fromString` is
the full text as it was before that edit. It is serialised as **Jira wiki markup** (`h2.`
headings, `*bold*`, `{{code}}`, `[text|url]`), not ADF and not markdown, so writing it back
through a markdown or ADF edit means converting the markup by hand. Convert it, show the restored
text in the plan like any other edit, and re-verify per H5.
