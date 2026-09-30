# Retro handovers: per-theme docs (on explicit ask only)

> Part of the `retro-processor` skill. Produced only on the operator's explicit ask, after the
> retro page exists and the themes are confirmed; see SKILL.md.

Produced **only when the operator explicitly asks**, after the retro page exists and the themes
are confirmed: never automatically, and never offered beyond the single reminder line in the
report. One doc per confirmed theme, seeding a fresh brainstorming session in a new agent
session. If the ask is partial ("just the ops one"), write only those docs.

Practicalities: save under the OS temp dir, never the workspace
(`$TMPDIR/retro-handovers-DD-MM-YYYY/handover-N-<kebab-theme>.md`, N in theme-table order); link
artifacts instead of duplicating them; redact anything sensitive (no secrets, no personal data);
tailor each doc to what its next session is *for*. Report the file paths back so each doc can
seed a fresh session.

Doc structure, nine parts in order:

1. **Title:** `# Handover N/M: <Theme>`.
2. **Next session:** one short paragraph: what the session is (usually a brainstorm, with the
   manager), and the named **deliverable** (e.g. "a model proposal to discuss with Alex, Sam and
   Robin, then the team"). A brainstorm's deliverable is a proposal, not an implementation.
3. **Where this comes from:** retro date; theme N of M with what it merges and its vote count;
   owner(s); a link to the retro child page; the transcript doc link **with the approximate
   timestamp ranges** of the relevant stretches (e.g. "roughly 00:47 to 00:55").
4. **What was said (condensed, with attribution):** bullets carrying who argued what, canonical
   roster spellings, short quotes where the phrasing matters. Condensed means condensed: the
   full record is the retro page; this section is what the brainstorm needs in its head.
5. **Current state and artifacts (link, don't duplicate):** the pages, boards, epics, channels
   and prior work the theme touches, by URL / ticket key, each with a half-line of why it
   matters. Cross-reference sibling handovers where themes feed each other.
6. **Constraints and landmines:** firm positions people hold, social constraints ("socialise
   before announcing"), and hard boundaries (e.g. org-level policy rules). These are the things
   a fresh session cannot infer and must not trample.
7. **Open questions to brainstorm:** numbered, concrete, answerable questions; this list is the
   next session's working agenda.
8. **Suggested skills:** the toolbox skills the next session should reach for, each with a
   one-line why.
9. **Conventions for outputs:** restate the standing rules the next session must honour (date
   format, formatting rules, where team-facing artifacts live, Slack drafts to the team channel
   with the manager sending by hand).
