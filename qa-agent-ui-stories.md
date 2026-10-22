# UI & Visualization — User Stories

*Continues the backlog. Stories 439–487. These are about layout, interaction, and what a persona sees — not about the data pipeline behind it. Each `Pass:` is an observable UI condition you can test by looking at the screen and clicking.*

**The model these are written against:** one shell, a persona toggle (Tester / Manager / Admin), a canvas of rearrangeable panels, a lens switcher that brings a panel forward, and a single drill-in drawer every panel zooms into. Build note for Cursor: ship the fixed per-persona defaults first (they're just the chosen layouts, arranged), then add lens-switching and drag-rearrange later. The composable engine can arrive after the defaults look right.

---

## AS. The shell & persona switch

**439. [P0]** As any user, I want the app to open on a layout that fits my role, so that I'm not handed a screen built for someone else's job.
`Pass:` A Tester lands in the three-pane workspace; a Manager lands on board + briefing; an Admin lands on briefing + cost + readiness — each without configuration.

**440. [P0]** As a user with more than one role, I want a visible persona toggle, so that I can move between altitudes without logging out or switching tools.
`Pass:` A Tester/Manager/Admin control is visible to multi-role users; switching re-lays the same shell rather than loading a separate app; single-role users don't see roles they can't use.

**441. [P1]** As a user, I want the switch to feel like the same product re-weighting, not a page reload, so that continuity is obvious.
`Pass:` Switching persona keeps the top bar, project context, and shell chrome in place; only the panel arrangement changes.

**442. [P1]** As a user, I want my current context (org, project, persona) always visible, so that I never act in the wrong place.
`Pass:` Active org, project, and persona are shown persistently; every action on screen applies to that context.

**443. [P1]** As a user, I want the shell to reuse the panes I already know, so that the tool reads as one system.
`Pass:` The left/center/right spatial logic of the tester workspace is recognisable in the manager and admin layouts — navigation on the left, focus in the center, detail on the right.

---

## AT. Panels, lenses & the modular canvas

**444. [P1]** As a manager or admin, I want each view to be a panel on a canvas, so that the layout is composed of parts rather than one fixed screen.
`Pass:` Briefing, board, timeline, cost, org map, and ROI each render as a self-contained panel with its own header and a way to expand it.

**445. [P1]** As a manager or admin, I want a lens switcher that brings a panel forward, so that I can focus without losing the others.
`Pass:` Selecting a lens from the top rail enlarges or foregrounds that panel; the selected lens is visibly marked; others remain reachable.

**446. [P2]** As a power user, I want to rearrange and resize panels, so that the layout matches how I work.
`Pass:` Panels can be moved and resized; the arrangement persists for that user and project across sessions.

**447. [P2]** As a power user, I want to add or remove panels, so that I only see what's relevant to me.
`Pass:` A panel can be hidden and re-added from a panel library; removing one never deletes its underlying data.

**448. [P1]** As a user, I want a way back to my role's default layout, so that experimentation is safe.
`Pass:` A "reset to default" control restores the persona's standard arrangement in one action.

**449. [P1]** As a user, I want panels to degrade gracefully at smaller widths, so that the view holds up on a laptop or a shared screen.
`Pass:` Below a breakpoint, panels stack in priority order rather than overflowing or clipping; nothing becomes unreachable.

---

## AU. The shared drill-in — one run, every altitude

> This is the spine. Everything zooms to the same atomic unit.

**450. [P0]** As any user, I want to click into a run's detail from anywhere it appears, so that I never hit a dead end.
`Pass:` A bar on the timeline, a card on the board, a node on the org map, and a number in the cost panel all open the same run-detail drawer.

**451. [P1]** As a user, I want the drill-in to open beside what I clicked, not replace it, so that I keep my place.
`Pass:` Detail opens in the right-hand drawer while the panel I came from stays visible; closing it returns me exactly where I was.

**452. [P1]** As a user, I want the run detail to be the same object at every altitude, so that the CEO and the tester are looking at one truth.
`Pass:` The run drawer shows identical core facts (target, verdict, evidence, timing) regardless of which persona or panel opened it.

**453. [P1]** As an admin, I want to zoom from an aggregate straight down to a single agent action, so that I can audit a surprising number.
`Pass:` An org-level figure drills through project → run → step → the specific agent action, each level navigable back up.

**454. [P2]** As a user, I want a breadcrumb of how deep I've drilled, so that I can climb back out cleanly.
`Pass:` The drill path is shown; clicking any level returns there without losing the trail.

---

## AV. Manager view — board, briefing & team

**455. [P0]** As a manager, I want a board of tasks by state, so that my team's work is visible at a glance.
`Pass:` Columns show task states (assigned, running, needs review, passed, blocked); each card shows owner, target, and a live status dot.

**456. [P0]** As a manager, I want a briefing feed of what needs me, so that I act on priorities instead of scanning everything.
`Pass:` The center panel lists the few things needing attention — runs to review, blocked testers, coverage drops — newest and highest-impact first.

**457. [P1]** As a manager, I want to see who is active right now, so that I know the shape of the team's day.
`Pass:` A team strip shows each tester and agent with an active/idle indicator and what they're currently on.

**458. [P1]** As a manager, I want to act on a card without leaving the board, so that triage is fast.
`Pass:` Approve, reassign, and comment are available from the card or its drawer; the card's state updates in place.

**459. [P1]** As a manager, I want the health numbers pinned above the board, so that status is always in view.
`Pass:` Pass rate, open criticals, coverage, and at-risk tasks sit in a strip that stays visible while I work the board.

**460. [P1]** As a manager, I want to drop into a tester's actual workspace from their card, so that I can see a run as they see it.
`Pass:` Opening a tester's active run shows the three-pane workspace view of it, then returns me to the board.

**461. [P2]** As a manager, I want overdue and blocked items to stand out visually, so that nothing rots silently.
`Pass:` Overdue and blocked cards carry a distinct treatment that reads at a glance without opening them.

---

## AW. Admin view — the executive briefing

**462. [P0]** As an admin, I want the landing view to state the answers in plain sentences, so that I get insight before charts.
`Pass:` The top of the admin view reads as statements — readiness, main risk, spend posture, one anomaly — each backed by a figure, not a bare chart grid.

**463. [P0]** As an admin, I want each statement to drill into its evidence, so that I can challenge it.
`Pass:` Clicking a briefing statement opens the chart or run set behind it.

**464. [P1]** As an admin, I want a clear release-readiness signal per product, so that go/no-go is answerable at a glance.
`Pass:` Each product shows one readiness state, visibly drillable into the tests behind it.

**465. [P1]** As an admin, I want stale or thin data marked, not shown as green, so that I never decide on bad information.
`Pass:` A readiness or metric past its freshness window is visibly flagged as stale; small samples are labelled rather than averaged silently.

**466. [P1]** As an admin, I want anomalies surfaced in the briefing, so that problems find me.
`Pass:` An unusual spike in defects, cost, or failures appears as its own briefing line without my going looking.

**467. [P2]** As an admin, I want to generate a shareable summary from the briefing, so that I can take it to the board unchanged.
`Pass:` An export produces the same statements as a clean, standalone summary.

---

## AX. Admin view — control centre lens

**468. [P0]** As an admin, I want a live control-centre lens, so that I can see the whole operation running in real time.
`Pass:` A dense panel shows active runs, agents deployed, tokens consumed, and models in use, updating live.

**469. [P1]** As an admin, I want spend broken down visually by project, team, and model, so that I can see where money goes without reading a table.
`Pass:` A breakdown (bars or a pie) shows cost share by dimension; selecting a slice drills into it.

**470. [P1]** As an admin, I want status indicators that read at a glance, so that a wall of tiles is scannable.
`Pass:` Healthy, warning, and failing states are distinguishable by treatment, not only by reading numbers; a legend is present where colour carries meaning.

**471. [P1]** As an admin, I want budget thresholds shown against actuals, so that overrun is visible before it happens.
`Pass:` Each budget shows consumed vs limit; approaching the cap is visually distinct from being under it.

**472. [P2]** As an admin, I want synthetic-monitor status lights in the control centre, so that production health sits beside team activity.
`Pass:` Monitored journeys show up/down/degraded status; a failing check is immediately obvious.

---

## AY. Admin view — org map & ROI lenses

**473. [P1]** As an admin, I want an org-map lens showing the hierarchy, so that I can see who owns what.
`Pass:` A tree or node map renders org → products → teams → members, each node coloured by health and navigable.

**474. [P1]** As an admin, I want nodes to reflect live activity, so that the map is a living view, not a static chart.
`Pass:` Nodes with running work are visibly distinct from idle ones; clicking a node drills into it.

**475. [P1]** As an admin, I want an ROI lens that opens on value delivered, so that I can justify the tool.
`Pass:` The ROI panel leads with hours saved, bugs caught before release, and cost per outcome — framed as outcomes, not usage.

**476. [P1]** As an admin, I want a before/after comparison, so that value is shown, not asserted.
`Pass:` A visual contrasts the pre-tool baseline with current performance on speed, coverage, and defects caught.

**477. [P2]** As an admin, I want an optional leaderboard lens, so that I can recognise output when I choose to.
`Pass:` The leaderboard is off by default, labels what it measures, and normalises for task complexity where possible — framed as throughput, not worth.

---

## AZ. Visual system, states & trust

**478. [P0]** As any user, I want one consistent visual language across all three personas, so that the tool feels like a single product.
`Pass:` Colour meanings (pass/fail/warning/blocked), status dots, and typography are identical across tester, manager, and admin views.

**479. [P1]** As any user, I want colour never to be the only carrier of meaning, so that the view is readable for everyone.
`Pass:` Every status conveyed by colour also carries a label, icon, or shape; the view survives a greyscale check.

**480. [P1]** As any user, I want empty states to guide setup, so that a new org or project isn't a wall of blank panels.
`Pass:` Before data exists, each panel explains what will appear and links the action that produces it, rather than showing zeros as results.

**481. [P1]** As any user, I want loading and live states shown honestly, so that I trust what I'm seeing.
`Pass:` Panels show a loading state while fetching and a clear "live" indicator when streaming; stale data is never shown as current.

**482. [P1]** As any user, I want the view to respect my role's boundaries visually, so that I'm never shown something I can't access.
`Pass:` Panels and drill-ins outside a user's permission are absent, not shown-then-blocked; a tester never sees org cost, a manager never sees projects they don't own.

**483. [P2]** As any user, I want the interface to work in light and dark mode, so that it fits the environment I work in.
`Pass:` Every panel, status colour, and chart is legible in both modes.

**484. [P2]** As a presenter, I want a clean full-screen view of any panel, so that I can show it in a meeting.
`Pass:` A panel can expand to a presentation-friendly view without surrounding chrome, and return cleanly.

---

## BA. Cross-persona flow (integration-level UI tests)

**485. [Flow]** As a manager reviewing a failure, I want to go board card → run detail → the tester's workspace view → back, so that I can investigate without losing my place.
`Pass:` The whole path works in both directions and returns to the original board scroll position.

**486. [Flow]** As an admin chasing a cost spike, I want to go briefing line → cost breakdown → project → run → agent action, so that I can find the cause of a number.
`Pass:` Each drill level opens correctly and every level is navigable back up to the briefing.

**487. [Flow]** As a user switching personas mid-task, I want the same run I had open to stay reachable, so that changing altitude doesn't lose my work.
`Pass:` A run open in the drawer remains accessible after a persona switch, presented at the new persona's altitude.

---

## Build order for Cursor

**Ship the three fixed layouts first (439, 455, 462).** The composable canvas is the eventual model, but a Tester workspace, a Manager board+briefing, and an Admin briefing are three concrete screens you can build and test now. Users never need to know the lens engine arrived later.

**Then the shared drill-in (450–452).** This is the single most important interaction in the whole enterprise layer — it's what makes three views feel like one instrument. Build it before the lens-switching, because every panel depends on it.

**Then role-boundary rendering (482).** It's a visual story but a security requirement: a panel a user shouldn't see must be absent, not rendered-then-hidden. Get this right before dashboards carry real cross-project data.

**Lenses and rearrange (444–447) come last.** They turn three good screens into one composable surface — high polish, low risk to defer. The product is fully credible with fixed layouts; the canvas is the upgrade.
