---
name: wrap-1-1
description: |
  Closes the loop on a 1:1 after the meeting: pulls the meeting transcript from
  your transcript source, completes your handwritten notes on the meeting page
  in your own terse style, closes or carries over the prep topics, drafts the
  shareable summary as a chat message you send by hand, logs observations into
  the person's private file, and marks the meeting Done. Use after a 1:1:
  "wrap up my 1:1 with Alex", "close the loop on Sam", "process Robin's
  transcript".
argument-hint: <name> [pasted transcript]
---

# Wrap 1:1: close the loop after a meeting

The counterpart to `prepare-1-1`. Everything it needs lives on the meeting page (briefing, prep
notes, your handwritten notes) plus the transcript, so it runs identically in the session that did
the prep or in a fresh one. No session state is assumed.

Write surface: the meeting page (summary block, prep-note dispositions, status), the person's local
`people/{slug}.md`, the optional `history_file`, confirmed glossary rows in `TEAM_CONTEXT.md`, and a
chat draft to the person. Nothing is written before the manager approves the wrap plan, and the chat
draft is never sent by the skill.

## Requirements

Degrade gracefully:
- `notes_tool` access is required (read and update the meeting page, set status). If unavailable,
  stop and say so.
- `transcript_source` access is only for auto-fetch. If unavailable, ask the manager to paste the
  transcript; don't stop the run.
- `chat_tool` draft support is only for the summary draft. If unavailable, output the shareable
  block in chat for manual copy-paste and carry on.

## Step 0: load team context

Read `TEAM_CONTEXT.md` at the repo root. Fields used: `reports_roster` (name to slug), `notes_tool`,
`meeting_title_convention`, `transcript_source`, `chat_tool`, `history_file` (optional),
`style_rules`. The person's private file is `people/{slug}.md` (local-only, gitignored, never
committed or shared).

## Step 1: Locate the meeting

Search the notes tool for `1:1 {FirstName}` (per `meeting_title_convention`); take the most recent
page with date on or before today and status not Done. If the best candidate is more than 7 days
old, or every recent page is already Done, list the candidates and ask; don't guess.

Read the full page and split it by the template's boundaries:
- **Private briefing** (above the first divider): open observations, agreed goals, development plan.
  Private context; informs judgment only.
- **Summary block**: from the `1:1 Summary: {date}` line down to the next divider. Contains the
  suggested questions and whatever the manager wrote during the meeting. This block is what gets
  shared with the report.
- **Prep notes** (below that divider): "Open Topics to Follow Up" / "Likely Closed Topics".

## Step 2: Get the transcript

- If a transcript was pasted with the invocation, use it as-is. Never override a paste with a
  search: an explicit paste is assumed intentional (e.g. an edited version).
- Otherwise search the `transcript_source` (e.g. Google Meet transcripts in Drive). Search loosely
  on the person's first name plus the meeting date, converted to the source's own date format. Known
  patterns worth checking:
  - Google Meet auto-transcripts: docs titled `{meeting name} - YYYY/MM/DD HH:MM TZ - Transcript`.
  - Gemini notes docs: `{meeting name} - YYYY/MM/DD HH:MM TZ - Notes by Gemini`, with the full
    transcript in a tab of the same doc. The title does NOT contain "Transcript", so also search by
    first name with a created-time filter for the meeting day.
  - Timing: these docs appear ~15 minutes after the meeting ends and start empty, filling in over
    the next few minutes. An empty read right after the meeting means wait and retry, not a missing
    transcript.
  - Language variants: one notes doc can exist per language, filling independently. One can be
    permanently truncated ("Transcription ended after HH:MM:SS") while a sibling holds the complete
    transcript. Read all sibling docs and retry before giving up; a complete transcript in any
    language beats a truncated one in another.
- More than one plausible match: list them and ask rather than guessing.
- No match (recorder not started, title drifted): say so and ask for a paste. Never fail the whole
  run over a missed auto-fetch.

From here on, a fetched and a pasted transcript are treated identically.

**Transcript hygiene:** follow the transcript hygiene conventions in `TEAM_CONTEXT.md`. Garble
candidates are proposed in the wrap plan (Step 5) and confirmed rows are appended in Step 6.

## Step 3: Complete the notes

Two evidence sources with different rules:

- **The manager's handwritten notes are first-class.** Never delete, reword, or reorder them.
  Managers sometimes note thoughts that were not said aloud; those stay (they are the manager's own
  notes, and the Step 5 plan flags every note-only item before it can reach the shared draft).
  Complete around them.
- **The transcript gets distilled, never quoted.** It is the report's verbatim words; the summary
  carries conclusions, not excerpts.

"Complete" means: add the discussion points, decisions, and action items the notes missed; attach
the outcome to a bullet the manager started; note who owns what if it was agreed. Nothing more.

**Boundary rule:** content from the briefing, the Observations Log, or reviews never enters the
summary block. That block gets shared. Same privacy rule as `prepare-1-1`.

## Tone: write like the manager, not like an AI

The summary block goes to the report under the manager's name. Before writing a word, fetch this
person's last 1-2 completed 1:1 pages and read their summary blocks; match that register exactly.

- Short bullets and fragments. No meeting-minutes prose.
- No intro or outro ("Great discussion about...", "We aligned on...", "Overall...").
- No invented headers, no bold topic labels, no emoji beyond what the manager already uses.
- If the manager mixes languages, keep the mix per item: a bullet started in one language is
  finished in it.
- Names, dates, numbers stay exact, never approximated.
- Apply the `style_rules` from TEAM_CONTEXT.md to all new text.
- A topic with nothing worth adding gets nothing. Silence over filler; fewer, denser bullets beat
  coverage.

Litmus test: the added bullets should read in the manager's voice, and the manager reviews every
one.

## Step 4: Close the loop on prep topics

Walk every item in "Open Topics to Follow Up" against transcript plus notes and assign a
disposition:
- ✅ closed: discussed and resolved
- 🔄 still open: discussed, not resolved
- ⏭ not raised: carry over to next time

Annotate the prep-notes section with these. The next prep reads the last 3 meetings, so the
dispositions are what feed forward.

## Step 5: The wrap plan (one document, before any write)

Assemble one consolidated plan: everything Step 6 will write, in final form. Each reviewable item
carries a short stable ID so a batch of suggestions can reference items tersely.

1. **Summary block** (`S1, S2, ...`, one ID per added bullet; the manager's own bullets get no ID,
   they are not up for review). Every note-only item stays flagged inline: "you wrote X; the
   transcript doesn't show it was said. Keep in the shared draft?"
2. **Chat digest** (`D1, D2, ...`), per the Step 6.2 rules.
3. **Topic dispositions** (compact table, rows `T1, T2, ...`).
4. **Candidate observations** (`O1, O2, ...`): new concerns or wins spotted in transcript or notes,
   formatted per the `log-observation` conventions, plus any `[open]` observations from the briefing
   that the conversation resolved (propose closing them).
5. **Candidate history events** (`H1, H2, ...`), if a `history_file` is configured: org events
   (departure, arrival, team move, role change, manager change, reorg, mentorship change) and
   relational events (collaboration, praise, friction, bridge) surfaced by transcript or notes.
   Relational events always carry attribution in their source field: who said it plus this meeting's
   page URL. Bar for extraction: would this change how you prep the other person's 1:1? Log only
   new-or-changed facts; an ongoing collaboration already in the history is not re-logged each
   meeting, but its start, a change, or its end is.
6. **Glossary rows** (`G1, G2, ...`): new garble candidates from Step 2, each with its source quote.
   Zero is a valid outcome.

Deliver the plan through the harness's plan feature when available (enter plan mode once Step 1 has
confirmed the meeting page; everything up to approval is read-only anyway). Feedback loops back as
plan revisions: apply the whole batch, update the plan, re-present. If a suggestion is ambiguous,
ask about that item inside the same revision, never as a separate exchange. Approval ends plan mode
and fires Step 6. Plan mode also enforces the invariant: nothing is written anywhere before
approval.

Fallback (plan mode unavailable, e.g. running as a subagent): present the same plan as one chat
message and run the identical loop by hand. The manager replies with one batch of suggestions (by ID
or free-form); apply all of them and re-show the full updated plan; rounds repeat until approval
("go"). The no-writes-before-approval rule still holds, by discipline instead of by the harness.

## Step 6: Run the plan (after approval)

Approval fires every write in one go, no per-write confirmations:

1. **Meeting page**: update the summary block and the prep-note dispositions. The manager's original
   text untouched.
2. **Chat draft to the person**: an outcome digest derived from the shared block, not a copy of it.
   Target 7-10 bullets, one screen on mobile, converted to your chat tool's formatting. Create it as
   a draft (e.g. a Slack draft DM); it is a draft, **never send it**. The manager reviews and sends
   it themselves. Digest rules:
   - Keep the title line `🚀 1:1 Summary: {date} 🚀` unchanged.
   - Merge each question and answer pair into one terse outcome bullet: the answer absorbed, the
     question dropped. No nested sub-bullets.
   - Drop entirely: questions that got no answer, template scaffolding, and context the person
     already has. An unanswered agenda item survives only if it carries forward as an action (e.g.
     "development plan main goal: top of next 1:1").
   - Keep exact: names, dates, numbers, owners of agreed actions.
   - Same register as the block: the manager's terse voice, language mix preserved per item,
     `style_rules` applied.
3. **`people/{slug}.md`**: append approved observations to `## Observations Log` (heading format
   `### {YYYY-MM-DD}: {title} {⚠️|✅|ℹ️} [open]`); flip resolved entries `[open]` to `[closed]` with
   a one-line conclusion; touch `## Patterns & Recurring Themes` only if something genuinely
   recurring emerged.
4. **History file**: append approved history events to the `history_file`, keeping it sorted
   ascending by date and syntactically valid.
5. **Glossary**: append approved glossary rows to the transcript-garble glossary in
   `TEAM_CONTEXT.md`.
6. **Meeting status: Done.**
7. Optionally spawn a background sweep with `intake-for-me sweep {page URL}` to pick up any inline
   capture markers the manager dropped during the meeting (if you use that skill).

## Output

Short report: link to the meeting page, N topics closed / M carried over, observation titles logged,
and where to find the chat draft. Don't restate the summary or digest in chat; both were shown in
the wrap plan.

## Usage

```
/wrap-1-1 <name> [pasted transcript]
```

- `<name>`: the report's name (required).
- A pasted transcript (optional) overrides the `transcript_source` search.

## Edge cases

- **Page already Done**: probably already wrapped; ask before re-processing.
- **Page not built by prepare-1-1** (no dividers): treat the whole page as your notes, skip topic
  dispositions, and say so in the plan.
- **No people file**: for a skip-level, create the minimal stub (same shape as `prepare-1-1`'s edge
  case) and log observations there; skip-level people's observations live in their own file, not
  their manager's. For a one-off attendee, skip step 6.3.
- **Transcript mangles names**: match speakers loosely; a 1:1 has two participants, so attribution
  works by elimination.
- **Transcript in another language or mixed**: fine; distill each topic in the language of the
  manager's notes for that topic.

## Privacy

- The transcript is the report's verbatim words: conclusions, not excerpts, go into the people file,
  the chat draft, and this chat.
- Knowledge-base content never appears below the first divider on the meeting page.
- The transcript stays in its source; don't copy it into the repo or onto the meeting page.
