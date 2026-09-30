# Follow-up action suggestions: signal catalogue

> Capability 3 of the `standup-processor` skill. The signal categories and detection cues used to
> build **up to 5** suggestions, plus the suggestion template. Suggestions are listed in the run
> report only; this skill executes none of them.

## Signal categories and detection cues

- **Documentation updates**: "finished/shipped/merged/deployed" suggests updating the relevant
  wiki page or release notes; "decided to / we agreed" suggests documenting the decision;
  "new process / from now on" suggests updating a runbook; new tooling or access changes suggest
  updating onboarding docs.
- **Jira housekeeping**: multiple tickets completed under an epic suggests reviewing the epic's
  status; "blocked by [other team]" suggests a Jira issue link; "turns out we also need"
  suggests a new ticket or estimate update; work mentioned that isn't tracked at all suggests a
  ticket (distinct from Capability 1's proposals: this catches things mentioned in passing or
  about someone else's work).
- **Slack outreach**: a blocker persisting across days or involving another team suggests
  messaging that team's channel or lead; decisions affecting absent stakeholders suggest posting
  more broadly; "great work by X" suggests public recognition; "anyone know how to" suggests
  posting in a relevant help channel.
- **Meeting follow-ups**: "let's take it offline / park that" suggests scheduling a follow-up or
  a discussion ticket; open questions left unanswered suggest an async follow-up; "can someone
  review" suggests tagging reviewers directly.
- **Risk flags**: the same blocker in consecutive stand-ups (needs prior-run context; a single
  transcript won't always show it) suggests escalation; "deadline is / running behind" suggests
  flagging in a status update; growing task mentions without ticket creation suggests a grooming
  review.

## Suggestion template

`[icon] **[Category]**: [Specific action] ("[Triggering quote from transcript]")`.
