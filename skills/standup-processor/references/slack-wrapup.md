# Slack wrap-up: detailed format

> Capability 2 of the `standup-processor` skill. The internal wrap-up's shape, section list, soft
> cap, and the one-draft-per-channel gotcha. Generic Slack markup rules are not repeated here:
> follow the Slack message formatting conventions in `TEAM_CONTEXT.md`.

## Structure

Draft a wrap-up for the team channel (`team_channel` in TEAM_CONTEXT.md), formatted
**epic-level first**: lead with what's closest to done and what needs help, not a per-person
recap. Structure:

- **What's moving**: one bullet per epic/theme with meaningful movement today (not every ticket).
- **Blockers / needs**: who needs what, called out explicitly.
- **New tickets & comments**: Capability 1's output, summarised (not the full table).

## List shape

**List shape** (a Slack rendering rule): title as `**Stand-up wrap-up: <date>**`, then the three
sections as bold headers with tight `-` bullets under each. Blank lines separate sections only,
**never bullets within a section**: a blank line between `-` items makes Slack parse each as its
own single-item list, stacking list margins on paragraph breaks, and the message sprawls over
multiple screens.

## Soft cap

**Soft cap: at most ~4 bullets per section, epic-level.** Overflow folds into one closing bullet
("plus N smaller moves: PROJ-101, PROJ-107") rather than growing the list; a section with
nothing to say is one short line ("none"), never dropped silently.

## Markup and pre-send self-check

Follow the Slack message formatting conventions in `TEAM_CONTEXT.md`.

**Pre-send self-check** before any draft call: no blank line between consecutive bullets;
titles and labels are `**...**`; no em dash; every ticket key is a link. Never prototype by
posting.

## One-draft-per-channel gotcha

Slack allows only one attached draft per channel. If a run covers more than one day (a catch-up
after a gap), do not create one draft per day: a later call silently overwrites the earlier
draft with no error, so only the last one survives. Build one combined draft with a clearly
dated section per day.
