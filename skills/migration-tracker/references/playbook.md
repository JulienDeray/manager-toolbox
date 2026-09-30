# Cross-Team Migration Playbook

> The methodology the migration-tracker skill operationalises: how a platform or infrastructure team
> runs the org-wide migrations it leads (deprecating a service, adopting a new platform, library or
> infrastructure upgrades that every other engineering team must adopt). Identifiers (register page,
> the engineering-teams roster) are not in this file; they live in `TEAM_CONTEXT.md`, loaded in
> Step 0.

## Why this exists

A platform team regularly leads migrations that other teams must adopt but the platform team does not
own the work for. Those teams have their own backlogs and priorities, so you must grant autonomy (each
team migrates at its own pace, on its own board) while keeping oversight and hitting a deadline. Left
ad-hoc, these projects stall: no single owner, no deadline, no visibility, no incentive to finish the
long tail. This playbook makes the method repeatable and gives it a harness: a Confluence register, a
Jira convention, the migration-tracker skill, and a weekly/monthly comms loop.

## The model: single owner, three stages

Every migration has one named owner (the driver, usually an engineer on your team) plus a backup owner
named from day one. The owner drives it through three stages; the Stage column on the register
reflects where it is. The hard deadline is backed by engineering leadership when it is announced (see
Deadlines).

1. **Derisk**: pilot with the 1 or 2 *hardest* teams first, not the easiest. Build the self-serve
   tooling, the runbook, and a validation path so teams can prove old and new behave the same. Most
   migrations fail here in hindsight: unclear funnels, unhandled edge cases, adoption resistance, not
   headcount.
2. **Enable**: docs plus tooling let teams self-migrate. This is the autonomy phase: each team creates
   its own tickets and runs at its own pace. Track progress; publish it; celebrate early finishers.
3. **Finish**: close the long tail. Freeze new work on the legacy thing in lagging teams near the hard
   deadline; the owner executes remaining edge cases directly rather than waiting.

### The long-tail problem (stalls at ~80%)

The easy 80% migrates fast; the last 20% is edge cases plus a motivation cliff. Counter it
deliberately: make progress visible (visible progress creates healthy accountability), keep unblock loops short (hours, not days),
freeze non-migration work in stragglers, and have the owner directly finish the final few.

### Carrot before stick

Lead with carrots: tooling, paired support, public recognition. Bring out the stick (CI blocks on new
usage, SLA or support withdrawal, "no new projects on the old thing") only late and always paired with
real help. Enforcement without support breeds resentment and stalls.

## Deadlines: soft, then hard

- **Soft deadline** first: a guidance target, no penalty. Its job is to surface who needs help versus
  who resists, and to give you data.
- **Hard deadline** second: legacy end-of-life, support ends, new usage blocked in CI. Announce it
  only once you have soft-deadline data and committed support, and with engineering leadership behind
  it.
- Every migration records both dates plus the owner, on the register and the tracking epic.

## Communication cadence

Consistency beats frequency; don't hide metrics; don't spam.

| Audience | Cadence | Surface |
|----------|---------|---------|
| Lagging or stalled teams | Bi-weekly nudge (default, configurable per migration) | Slack draft in the team's channel; the manager sends it from Slack |
| Your own team (internal visibility) | Weekly | A "migrations needing attention" section in your planning agenda |
| Engineering leadership | Monthly | The leadership digest (% complete, forecast-to-deadline, escalations), a Slack draft the manager sends |

The register's Last comms / Next nudge due columns are the memory: a nudge is due when
`today >= last comms + cadence`. Nudge only the teams that are behind or stalled, never everyone.

## Tracking: Jira (live execution) + Confluence (governance)

Complementary, not either/or. Jira holds what each team is actually doing; Confluence holds the
portfolio view leadership reads.

### Jira convention

- One tracking epic per migration, in your team's project, carrying your project work-type label plus
  `migration::<slug>`. Owner = assignee; backup owner and both deadlines in the description.
- Per-team work associates back via one of two mechanisms (the status rollup keys off either):
  - **Epic-parent**: the team's ticket is parented to your tracking epic. Preferred where it works.
  - **Label + issue-link**: the team's ticket carries `migration::<slug>` and a "relates to" link to
    the epic. The universal fallback, and the only option for some teams (below).
- The expected-teams list (the denominator for % complete) lives on the migration's detail page and
  in the epic description. A team with no associated ticket = not started.

#### Cross-project parenting: a hard Jira constraint

- **Company-managed** ("classic") projects can parent their issues under an epic in another
  company-managed project, so those teams can parent tickets under your tracking epic.
- **Team-managed** ("next-gen") projects cannot parent their issues to an epic in another project at
  all. For those teams the `migration::<slug>` label + issue-link is mandatory, and it is the safe
  default everywhere. Onboarding should never assume universal parenting; record each team's managed
  type in the roster (`engineering_teams_roster` in TEAM_CONTEXT.md).
- A zero-cost live confirmation (drag a throwaway issue under a cross-project epic in the Jira UI)
  can be run against a representative target project before the first real onboarding.
- Some Jira sites have a dedicated child-level "Migration" issue type. It can be handy for your own
  project's migration tasks, but it is not an epic and will not exist in other teams' projects, so it
  is never the cross-team association mechanism; the tracking epic + label/link is.

### Confluence register

One register page (id in TEAM_CONTEXT.md as `migrations_register_page`) is the single portfolio source
of truth. One row per active migration:

**Migration · Owner · Stage** (Derisk / Enable / Finish) **· Soft deadline · Hard deadline ·
Progress** (teams done / total) **· Last comms · Next nudge due · Status** (On track / At risk /
Stalled / Done) **· Epic**.

Larger migrations also get a detail page (owner + backup, why, the expected-teams checklist,
deadlines, links to design doc / runbook / tooling, FAQ, and the comms intake log).

## Failure-mode checklist (review at onboarding and at each scan)

| Failure mode | Guard |
|--------------|-------|
| Stalls at ~80% | Derisk with the *hardest* teams; build validation tooling; freeze + owner finishes the tail |
| No clear owner | One named owner (+ backup) per migration, recorded on the register |
| No or soft-only deadline | Soft target first, then a hard deadline with leadership backing and CI/SLA teeth |
| Poor visibility | Weekly agenda section + monthly leadership digest; never hide metrics |
| Wrong design found mid-flight | If multiple teams hit the same blocker, the design may be wrong; listen and pivot fast |
| Recognition all front-loaded | Celebrate completion, not kickoff |
| Owner leaves | Name a backup owner from day one |

## Sources

Will Larson: [Migrations: the sole scalable fix to tech debt](https://lethain.com/migrations/),
[Your migration probably isn't failing due to insufficient staffing](https://lethain.com/migration-isnt-failing-due-to-lack-of-staffing/);
Monzo: [How we run migrations across 2,800 microservices](https://monzo.com/blog/how-we-run-migrations-across-2800-microservices);
Stripe: [Online migrations at scale](https://stripe.com/blog/online-migrations);
Datafold: [Stuck at 80%? The long-tail problem](https://www.datafold.com/blog/80-percent-done-migration-trap);
Google: [Software Engineering at Google: Deprecation](https://abseil.io/resources/swe-book/html/ch15.html);
Ad Hoc: [The carrots and sticks of platform governance](https://adhoc.team/2022/06/30/carrots-sticks-of-platform-governance/).
