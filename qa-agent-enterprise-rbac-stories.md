# Enterprise Layer — Roles, Management & Executive Visibility

*Continues the backlog. Stories 376–438. Covers organization structure, simple RBAC, team invites, and the two personas above the tester: the Project Manager who runs the team, and the Super Admin (CTO/CEO) who makes decisions from what the team produces.*

---

## The idea

Three personas, three definitions of value:

| Persona | What they open the tool to do | What they're really buying |
|---|---|---|
| **QA Tester** | Run tests, find bugs, build suites | A faster way to do their job (everything built so far) |
| **Project Manager / QA Lead** | Assign work, track quality, manage tickets | Control and visibility over a team's output |
| **Super Admin (CTO / CEO)** | See the whole org, spend, and readiness | Confidence in decisions and proof of ROI |

The tester layer is done. This doc is the two layers above it. One nudge for build order, since you're letting Cursor make the structural calls: **model permissions as capabilities, not hard-coded roles.** Three roles today, but if an enterprise asks for a fourth ("read-only auditor," "security reviewer"), that should be configuration, not a rewrite. Everything below assumes that.

---

## AI. Organization, projects & membership

**376. [P0]** As an admin, I want to create an organization that everything else lives under, so that my company's data is isolated from every other customer's.
`Pass:` All projects, users, runs, and usage belong to one org; nothing is visible or countable across org boundaries.

**377. [P0]** As an admin, I want to create projects inside the org, so that work is separated by product, team, or client.
`Pass:` Each project has its own members, tests, artifacts, and verified targets; a user in one project sees nothing in a project they're not a member of.

**378. [P1]** As an admin, I want a user to belong to multiple projects with a different role in each, so that one person's access matches their real responsibilities.
`Pass:` A user can be a tester in Project A and a manager in Project B; their view changes per project accordingly.

**379. [P1]** As an admin, I want org-level settings that all projects inherit, so that standards are set once.
`Pass:` Default SLOs, security policies, model choice, and verified domains set at org level apply to new projects unless overridden.

**380. [P1]** As a member, I want to switch between projects and orgs cleanly, so that I always know which context I'm acting in.
`Pass:` The current org and project are always visible; actions apply only to the active context; switching never leaks data from the previous one.

**381. [P2]** As an admin, I want to archive a project without deleting its history, so that finished work stays auditable.
`Pass:` Archived projects are read-only, excluded from active dashboards, and fully restorable.

---

## AJ. Roles & permissions (simple RBAC)

**382. [P0]** As an admin, I want three built-in roles — Tester, Manager, Super Admin — so that access matches responsibility out of the box.
`Pass:` Tester operates within assigned projects; Manager additionally assigns work and sees team dashboards for their projects; Super Admin sees and controls everything org-wide.

**383. [P0]** As an admin, I want each role to have a clear ceiling on what it can see and do, so that people can't reach beyond their remit.
`Pass:` A tester cannot see another project's data, another user's cost, or org-wide settings; a manager cannot see projects they don't own; each boundary is enforced server-side, not just hidden in the UI.

**384. [P0]** As a tester, I want to see exactly what is tracked about me, so that visibility feels fair rather than like surveillance.
`Pass:` A tester can view their own activity, usage, and metrics — the same data their manager sees about them — and the product is transparent that this is collected.

**385. [P1]** As an admin, I want to change someone's role and have it take effect immediately, so that access reflects reality.
`Pass:` A role change updates permissions on the next action; in-flight work isn't corrupted; the change is logged.

**386. [P1]** As an admin, I want a user's own runs and artifacts to survive a role change or removal, so that the team's work isn't lost with the person.
`Pass:` Demoting or removing a user leaves their tests, suites, and reports intact and reassignable.

**387. [P2]** As an admin, I want role changes and permission checks recorded, so that access is auditable.
`Pass:` Every grant, change, and denied action is logged with identity, timestamp, and context.

**388. [P2]** As an admin, I want the ability to add a custom role later without re-architecting, so that enterprise-specific needs are met.
`Pass:` A new role can be defined as a set of capabilities through configuration; existing roles are unaffected.

---

## AK. Invites, onboarding & offboarding

**389. [P0]** As a manager, I want to invite people by email to a project with a role, so that my team can start using the tool.
`Pass:` An invite email arrives; accepting places the user in the right project with the right role; declining or ignoring leaves no access.

**390. [P1]** As an admin, I want invites to respect a seat limit, so that billing stays predictable.
`Pass:` Inviting beyond the seat count is blocked with a clear message; freeing a seat re-enables invites.

**391. [P1]** As a manager, I want to see pending, accepted, and expired invites, so that onboarding doesn't fall through cracks.
`Pass:` Invite status is visible; expired invites can be resent; a resend doesn't create a duplicate membership.

**392. [P1]** As an admin, I want an invite locked to the intended person and org, so that a forwarded link can't grant access to someone else.
`Pass:` An invite accepted from a different email or after expiry is rejected; it can't be replayed to join a different org.

**393. [P0]** As an admin, I want to remove someone and cut all their access at once, so that offboarding is clean and safe.
`Pass:` Removal ends active sessions, revokes project access across the org, and prompts me to reassign their open work — while preserving their completed work.

**394. [P2]** As an admin, I want SSO group membership to map to roles, so that access follows our identity provider.
`Pass:` A user's group in the IdP determines their role; removing them there removes them here.

---

## AL. Project Manager — assigning and managing work

**395. [P0]** As a manager, I want to assign a testing task to a tester, so that work is distributed deliberately rather than ad hoc.
`Pass:` A task with a target, scope, and goal is assigned; the tester sees it in their queue; the manager sees its status.

**396. [P0]** As a manager, I want to track every task through its lifecycle, so that I always know what's in progress, blocked, or done.
`Pass:` Each task shows a clear state (assigned, running, needs review, passed, failed, blocked) and who owns it.

**397. [P1]** As a manager, I want to review and approve what a tester or agent produced, so that quality is gated before it counts as done.
`Pass:` A produced suite or finding can be approved, sent back with comments, or rejected; the state reflects the decision.

**398. [P1]** As a manager, I want to convert a bug or requirement into an assignable task, so that work flows from findings without leaving the tool.
`Pass:` A finding or a BRD requirement becomes a task with context attached, assignable in one action.

**399. [P1]** As a manager, I want to reassign or rebalance work, so that a bottleneck on one tester doesn't stall a release.
`Pass:` Tasks reassign with their context intact; the old and new owners are both notified.

**400. [P1]** As a manager, I want two-way sync with our issue tracker, so that the tool fits how the team already works.
`Pass:` Tasks and findings sync with Jira/Linear/GitHub Issues without duplication; status changes flow both directions.

**401. [P2]** As a manager, I want to set priorities and due dates, so that the team works on what matters first.
`Pass:` Priority and deadline are visible on tasks and drive sorting and overdue flags.

**402. [P2]** As a manager, I want to schedule recurring test runs for the team, so that regression coverage doesn't depend on someone remembering.
`Pass:` A scheduled suite runs and reports without manual initiation; failures notify the owner.

---

## AM. Project Manager — quality status & team performance

**403. [P0]** As a manager, I want a dashboard of my project's quality status, so that I can answer "are we okay to ship" without chasing people.
`Pass:` One view shows test pass rate, open defects by severity, coverage, and trend since last release.

**404. [P0]** As a manager, I want to see what each tester is working on and how it's going, so that I can manage without micromanaging.
`Pass:` A per-tester view shows current tasks, throughput, and blockers — surfaced as workload, not judgement.

**405. [P1]** As a manager, I want coverage mapped to requirements, so that I can see what's tested and what's exposed.
`Pass:` A traceability view shows covered, partially covered, and untested requirements for the project.

**406. [P1]** As a manager, I want defect trends over time, so that I can tell whether quality is improving or slipping.
`Pass:` Defects opened vs resolved, and where they cluster, trended across releases.

**407. [P1]** As a manager, I want to see where the agents are spending effort, so that I can redirect it toward risk.
`Pass:` A breakdown of runs by test type and area shows whether effort matches where bugs actually are.

**408. [P1]** As a manager, I want alerts when something needs my attention, so that I'm not refreshing a dashboard.
`Pass:` Configurable alerts fire on a failed release-critical run, an overdue task, or a spike in defects — delivered where the team already talks.

**409. [P2]** As a manager, I want a shareable status report generated for me, so that I can update stakeholders without assembling it by hand.
`Pass:` A plain-language summary of quality status is produced on demand or on a schedule.

**410. [P2]** As a manager, I want the empty state to be useful on day one, so that a new project doesn't show dead dashboards.
`Pass:` Before data exists, the view guides setup and shows what will appear, rather than blank panels.

---

## AN. Super Admin — the whole organization at a glance

**411. [P0]** As a super admin, I want a single view of every project, team, and their status, so that I can see the health of the whole operation.
`Pass:` An org-wide view lists all projects with quality status, activity, and readiness, drillable into any one.

**412. [P0]** As a super admin, I want the org's structure as a hierarchy, so that I understand who owns what.
`Pass:` A tree or map shows org → projects → members and their roles, navigable and current.

**413. [P1]** As a super admin, I want to see how testing effort is distributed across the org, so that I know where quality investment is going.
`Pass:` A breakdown (e.g. a pie or bar) shows share of activity by test type, by project, and by team — "most effort went to UI and regression testing this quarter."

**414. [P1]** As a super admin, I want a leaderboard of team and individual output, so that I can recognise and compare productivity.
`Pass:` A ranked view of throughput exists, normalised enough to be fair, with a clear note on what it measures.
*Design note: leaderboards cut both ways — some buyers love them, some QA cultures revolt at being ranked, especially since a tester on a hard app looks "slower" than one on a simple form. Make it optional, normalise for task complexity where you can, and frame it as throughput, not worth. Sell it as available, not default-on.*

**415. [P1]** As a super admin, I want to compare projects and teams on the same measures, so that I can spot both the models to copy and the areas that need help.
`Pass:` Projects are comparable on coverage, defect rate, cycle time, and spend, side by side.

**416. [P1]** As a super admin, I want the org-wide view to answer executive questions directly, so that I get insight, not just charts.
`Pass:` The view surfaces plain statements — "release readiness is 82% across active products; Payments is the main risk" — not only raw metrics.

**417. [P2]** As a super admin, I want anomalies surfaced proactively, so that problems find me rather than the reverse.
`Pass:` Unusual spikes in defects, cost, or failure rates are flagged without my going looking.

---

## AO. Super Admin — usage, cost & the control centre

> This is the section that closes the deal. A CTO signs when they can see and control spend.

**418. [P0]** As a super admin, I want a live view of usage across the org, so that I know what's being consumed in real time.
`Pass:` A control-centre view shows active runs, agents deployed, tokens consumed, and models in use, updating live.

**419. [P0]** As a super admin, I want spend attributed to project, team, and person, so that I know where the money goes.
`Pass:` Cost breaks down by project, by user, and by model, over any time window.

**420. [P0]** As a super admin, I want to see which models are used for what, so that I can govern the inference mix.
`Pass:` Usage shows which model handled which kind of work, with the cost of each — "most spend is on high-reasoning runs for exploratory testing."

**421. [P1]** As a super admin, I want budgets and caps per project or team, so that spend can't run away.
`Pass:` A limit triggers alerts as it's approached and can throttle or halt activity at the ceiling.

**422. [P1]** As a super admin, I want cost attributed correctly for unattended runs, so that scheduled and CI-triggered activity isn't a blind spot.
`Pass:` A run with no human at the keyboard is attributed to the project and the schedule/trigger that launched it, not left uncounted.

**423. [P1]** As a super admin, I want to forecast spend from current trends, so that I can budget ahead.
`Pass:` A projection estimates the period's spend from usage so far, with the main drivers named.

**424. [P1]** As a super admin, I want to see efficiency, not just consumption, so that I can tell wise spend from waste.
`Pass:` Cost is shown against output — cost per test, per bug found, per release verified — so spend maps to value.

**425. [P2]** As a super admin, I want to set org-wide model and cost policy, so that governance is centralised.
`Pass:` A default model and a cost policy set at org level apply everywhere unless a project is explicitly allowed to override.

---

## AP. Super Admin — release readiness & the development picture

> High-level only. A CEO wants "can we ship," not a diff.

**426. [P0]** As a super admin, I want a release-readiness signal per product, so that go/no-go is grounded in evidence.
`Pass:` A clear readiness state per product combines test pass rate, open critical defects, and coverage into a single, drillable verdict.

**427. [P1]** As a super admin, I want readiness tied to what changed via GitHub, so that I know the tested build matches the shipping build.
`Pass:` Readiness references the commit/PR/release it was measured against; a note flags when new changes have landed since testing.

**428. [P1]** As a super admin, I want to know when a readiness signal is stale, so that I don't ship on old information.
`Pass:` A readiness state past its freshness window is marked stale, not shown as green.

**429. [P1]** As a super admin, I want a high-level testing-vs-development view, so that I can see whether QA is keeping pace with shipping.
`Pass:` A simple view shows features shipped vs tested and the gap, without development-level detail.

**430. [P2]** As a super admin, I want release readiness trended across releases, so that I can see if quality discipline is holding.
`Pass:` Readiness at each release is tracked over time.

---

## AQ. Super Admin — tool performance, synthetic monitoring & ROI

> Not just how the *team* performs — how the *tool* performs for them. This is the ROI story that justifies renewal.

**431. [P1]** As a super admin, I want synthetic monitoring of key journeys running continuously, so that I know the product is healthy in real environments, not just at test time.
`Pass:` Scheduled synthetic checks run against monitored journeys and report uptime, latency, and failures over time.

**432. [P1]** As a super admin, I want to be alerted when a synthetic check breaks, so that I hear about production issues early.
`Pass:` A failing synthetic journey raises an alert with evidence, before or alongside customer reports.

**433. [P1]** As a super admin, I want to see how efficiently the tool itself is operating, so that I know it's optimising, not just consuming.
`Pass:` A view shows token efficiency, cache/reuse savings, and how spend per unit of output is trending — the tool earning its keep.

**434. [P0]** As a super admin, I want a clear ROI picture, so that I can justify the contract internally.
`Pass:` A view translates activity into business terms — tester hours saved, bugs caught before release, cycle-time reduction, cost per outcome — against a stated baseline.

**435. [P1]** As a super admin, I want to compare against how we tested before the tool, so that value is provable, not asserted.
`Pass:` A before/after comparison shows the delta in speed, coverage, and defects caught.

**436. [P2]** As a super admin, I want an exportable executive summary, so that I can take the ROI story to my board.
`Pass:` A clean, shareable summary of value delivered is generated on demand.

---

## AR. Cross-cutting edge cases

**437. [P1]** As an admin, I want every dashboard to respect role boundaries, so that no view leaks data a person shouldn't see.
`Pass:` A manager's org query returns only their projects; a tester never sees org cost; aggregate views never expose another project's detail. Enforced in the data layer, not just the UI.

**438. [P1]** As a super admin, I want aggregate views to hold up when data is thin, mixed, or stale, so that dashboards stay honest.
`Pass:` Small samples are labelled as such; incomplete or stale data is marked, not silently averaged into a misleading number; empty orgs guide setup rather than showing zeros as if they were results.

---

## Where to focus for the deals on your list

**The demo that closes is AO + AQ, not the test features.** Your buyer has seen testing tools. What they haven't seen is a live control centre showing spend by team and model (418–420), sitting next to an ROI view that says "this saved 340 tester-hours and caught 28 release-blocking bugs last quarter" (434). That's the slide that gets you the second meeting with procurement.

**Release readiness (426) is the CEO's single question.** Everything else is supporting evidence for "can we ship, and do we know why." Make that one signal excellent and drillable.

**Be careful with the leaderboard (414).** You asked for it and some buyers will love it, but ranked "productivity" is the fastest way to make a QA team quietly sabotage a rollout — and rollout resistance is what kills enterprise deals after signature. Ship it optional, complexity-normalised, and framed as throughput. Let the buyer turn it on; don't make it the default face of the tool.

**Get the data boundary (437) right before you build a single dashboard.** One cross-project leak in a security review and the whole enterprise conversation ends. Role-scoped data access is the foundation every view above sits on.
