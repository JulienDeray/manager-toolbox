# Retro synthesis: from board + transcript to the page's content

> The gather/analyse half of the `retro-processor` skill: what to extract from the two inputs
> and how to consolidate it. Runs entirely in plan mode, before any write. Its output feeds the
> checkpoint (SKILL.md, Flow step 3) and the page build
> ([page-structure.md](page-structure.md)).

## Contents

- Inputs contract
- Transcript handling
- Extraction rules
- Action-item consolidation (the 3-4 tops rule)
- Garble decodings: proposal format
- Checkpoint contract

## Inputs contract

- **Board export** (required): pasted markdown from the retro board. A typical shape: four
  columns (*What went well / What went less well / What do we want to try next / What puzzles
  us*) with grouped stickies and dot-votes. Treat any given shape as a worked example, **not a
  schema**: accept column-name and column-count variation, grouped or ungrouped stickies.
  Whatever came in is reproduced **verbatim** in the page's Board snapshot section.
- **Transcript** (required unless the operator explicitly opts into board-only): the meeting
  notes-plus-transcript doc, via pasted link or the Step 0a search.

## Transcript handling

- Cache the doc's full text to the session scratchpad, then read it **in full, in sequential
  chunks**. The AI summary/notes block at the top is never sufficient on its own: the raw
  transcript is the authority for what was actually discussed, who said it, and what was
  actually agreed.
- **Follow the transcript hygiene conventions in `TEAM_CONTEXT.md`** while reading: decode
  against the Step-0 glossary, and record each new garble candidate with its surrounding quote;
  they become proposals (format below), not silent corrections.

## Extraction rules

- **Summary**: a few sentences: the period's wins, the period's pains, and the biggest topic by
  votes. Lead with the verdict on the period's central question when there is one (e.g. "the
  new on-call model is working").
- **Main topics discussed**: only topics that got **real discussion time** in the transcript,
  not every sticky. One heading per topic with vote count where known; each a 2-6 sentence
  synthesis with **attribution** (who argued what, roster spellings), ending with any alignment
  reached or action taken. Order: votes first, then discussion order.
- **Decisions**: only alignments **actually reached in the session** ("direction agreed", an
  explicit adoption). Aspirations, options floated, and "we should look into" items are topics
  or action items, not decisions.
- **Attendance**: derive from the transcript's participants against the Step-0 roster (e.g.
  "10 in the call, 3 absent"); ask the operator if the count is unclear.

## Action-item consolidation (the 3-4 tops rule)

**Hard rule: consolidate action themes to 3, 4 tops.** It is a cap, not a target: 2 strong
themes beat 4 padded ones.

- **Merge related clusters** into one owned theme. Worked example: an on-call evolution sticky
  cluster + a shared-ops-handle decision + an access-ownership question merge into one theme,
  "Distributed ops & ownership".
- **Votes rank themes, they don't multiply them**: the top-voted topic leads the table; a
  5-vote cluster that belongs inside a bigger theme gets merged, not promoted.
- **Individual engineers' work commitments are never themes.** They stay on the page under
  **Small one-offs** (a task list with checkboxes, one per person + commitment).
- **Voted but not deep-dived** topics go under **Parked**, with an explicit "revisit at the
  next retro" note: visible, but no owner and no handover.
- Each theme row: **#** · **Theme** (with vote count where known) · **Owner(s)** (can be
  several, e.g. "Alex, with Sam and Robin") · **Next step** (one concrete sentence, not a
  restatement of the theme).

## Garble decodings: proposal format

For each new garble candidate, one proposal row:

| Garbled as | Proposed decoding | Evidence |
|-----------|-------------------|----------|
| "Cooper Netties operator" | **Kubernetes operator** (the cluster-tooling work track) | "...we should get the Cooper Netties operator running on staging..." |

Every proposal is **individually confirmed at the checkpoint**. Confirmed rows become glossary
edits in the plan; unconfirmed or rejected ones are dropped and flagged in the final report. A
decoding is never written on the skill's own judgement.

## Checkpoint contract

Before presenting the write plan, put to the operator as an explicit question step (not buried
in prose):

1. The **theme table** (# · Theme · Owner(s) · Next step) plus the Parked and one-off lists, and
   the proposed child-page short title.
2. The **garble proposals**, one confirm each.

Nothing is final until this passes: operator edits (split a theme, re-own it, rename the retro)
fold back into the synthesis, and only then is the write plan presented for approval. If the
operator and the transcript disagree, the operator wins; note the divergence on the page only if
they ask.
