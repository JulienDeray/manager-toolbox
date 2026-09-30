---
name: retro-processor
description: |
  Wraps up the team's periodic retrospective. Reads the retro board export and the meeting
  transcript in full, consolidates action items into 3-4 owned themes, and writes one durable
  record: a Confluence child page, a row in the Retrospectives index, confirmed glossary rows, and
  a Slack wrap-up draft the manager sends by hand. Themes and every write are confirmed with the
  operator first; handover docs only on explicit ask.
  Use when wrapping up or processing a retro, retro notes, or the retro board, or when the user
  mentions the retrospective wrap-up or retro action items.
---

# Retro Processor

The **wrap-up** of the team's periodic retrospective. It encodes a settled ritual shape rather
than redesigning it each run. It is the retro-cadence sibling of `planning-processor` and
`stepback-processor`; unlike the step-back it does **not** invoke planning-processor: the retro
is its own session, not a planning replacement.

Exactly **four writes**, all previewed in one plan: the retro's **child Confluence page**, one
**row in the Retrospectives index table**, new rows in the **transcript-garble glossary**, and
the **Slack wrap-up draft**.

Scope fences (deliberate scope decisions):

- **Wrap-up only.** No retro preparation: no agenda, no reminder, no scheduling. If preparation
  is ever wanted, it is a new capability (or a planning-agenda hook), not an extension of this
  flow.
- **No Jira tickets from action items.** Brainstorming comes first; themes flow into handover
  docs on explicit ask (below), never onto the board from this skill.
- **Handovers are never automatic.**

## Requirements

- **Confluence MCP tools**: get page, create page, get page descendants (update page only as a
  guarded fallback; see [references/wrapup.md](references/wrapup.md)).
- **Confluence REST v2** with an API token (e.g. email + token in a repo-root `.env`), needed
  only for the index-row append: the MCP HTML round-trip drops the index's `ac:link` bodies (why,
  and the fallback ladder: [references/wrapup.md](references/wrapup.md)). The glossary edit is a
  local file edit, unless you keep the glossary on Confluence.
- **Slack MCP tools**: the draft tool. If it is unavailable, the text in chat is the
  deliverable; this skill never calls the send tool.
- **A transcript source** (soft): a Drive/file-search MCP for the transcript auto-fetch
  (Step 0a); degrade to asking for a link or a paste.
- **Plan mode** plus an in-chat question step for the theme/decoding checkpoint.

No Jira writes, no board-tool API calls. If a required tool is missing, say so and stop rather
than working around it.

## Step 0: load team context

Read **`TEAM_CONTEXT.md`** at the repo root. This skill needs:

- `retro_index_page`: the Retrospectives index page id. If the field is absent, **stop**: this
  skill is meaningless without the index page. Point the operator at the bootstrap: create the
  index per [references/page-structure.md](references/page-structure.md) and register its id in
  TEAM_CONTEXT.md.
- `team_channel`
- `team_roster` (canonical name spellings for attribution; headcount for the attendance line)
- the **transcript-garble glossary** section (a table mapping garbled strings from
  auto-transcripts to their decoded meanings), in full. It is used twice: decoding while reading
  the transcript, and maintenance at the end of the run.
- `date_format` and `transcript_source`

Any other configuration-load failure is also a hard stop: say which resource failed, do not
proceed with partial configuration.

## Step 0a: transcript (when no doc link pasted)

The retro is usually a recorded meeting whose notes-plus-transcript doc (e.g. Gemini notes from
a Google Meet) lands in Drive shortly after it ends. If no link was pasted, **search the
transcript source before asking**: match loosely on the meeting title plus the retro date (a
hedge against title drift), and list the candidates if more than one matches. A pasted link
always wins over a search. Then fetch the doc, cache the full text to the session scratchpad,
and **read it in full, in sequential chunks** (these docs run tens of thousands of characters).
Never synthesize from the AI summary alone: the raw transcript is the authority for what was
*actually discussed*. If no doc is found, ask for a link or a paste; never proceed without the
transcript unless the operator explicitly opts into a board-only wrap-up.

## Flow: plan, approve, execute, verify

Plan mode, then an operator checkpoint, then approval, execution, and verify-reads. Synthesis rules:
[references/synthesis.md](references/synthesis.md); write mechanics:
[references/wrapup.md](references/wrapup.md); page contract:
[references/page-structure.md](references/page-structure.md).

1. **Gather (plan mode).** Require the board export (ask if absent); get the transcript (paste
   or Step 0a) and read it in full. Read the Retrospectives index page via a REST storage GET
   (one read serves the idempotency check and the later splice). **Idempotency:** if a child
   page or an index row already carries this retro's date, stop and ask; the retro was likely
   already processed.
2. **Synthesize** per [references/synthesis.md](references/synthesis.md): summary, main topics
   *actually discussed* (attributed via the roster), aligned decisions, and action items.
   **Hard rule: consolidate action themes to 3-4 tops.** Individual engineers' work commitments
   stay on the page as one-offs, never as themes; voted-but-not-deep-dived topics are parked.
   Collect candidate garble decodings while reading (follow the transcript hygiene conventions
   in `TEAM_CONTEXT.md`).
3. **Checkpoint (before presenting the write plan).** Confirm with the operator, as an explicit
   question step: the **theme table** (# · Theme · Owner(s) · Next step) and **each proposed
   garble decoding** with its source quote. Unconfirmed decodings are dropped from the plan,
   flagged in the report, never written. Theme edits fold back into the synthesis before the
   plan is presented.
4. **Approve.** The plan shows: the exact child-page title (`Retro DD-MM-YYYY: <short title>`)
   and section skeleton, the exact index-row cells, the exact glossary rows, and the **full
   Slack draft text**. Approval authorises exactly the four writes and nothing else: explicitly
   no Jira writes and no handover docs.
5. **Execute + verify.** The create/splice/PUT mechanics and the mandatory verify-reads:
   [references/wrapup.md](references/wrapup.md).
6. **Slack draft.** Created automatically once the plan is approved, with no extra confirm step:
   a draft attached to the team channel via the draft tool; the manager reviews and sends by
   hand. Never call the send tool.
7. **Report.** Child-page and index links, glossary rows landed (or "no new garbles"), the
   draft's location, plus one line reminding that per-theme handover docs are available on ask.
   Only claim success after the verify-reads match the approved plan.

## Handovers (on explicit ask only)

Produced **only when the operator explicitly asks**, after the retro page exists and the themes
are confirmed: never automatically, and never offered beyond the single reminder line in the
report. One doc per confirmed theme, saved under the OS temp dir (never the workspace), each
seeding a fresh brainstorming session.

**Full procedure:** [references/handovers.md](references/handovers.md).

## Usage

After the retro session, with the board export pasted:

```
/retro-processor <pasted board export> [transcript doc link]
```

The skill loads config, finds and reads the transcript, synthesizes, confirms themes and
decodings, presents the four writes for approval, executes, verifies, and leaves the Slack draft
for the manager to send. Ask afterwards for the per-theme handover docs if you want them.
