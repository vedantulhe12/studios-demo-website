# Campaigns, envelopes and approvals

How a request like "regression-test release 4.2 across three locales" becomes work that runs inside a spending
envelope, asks for exactly the signatures it needs, and never loses what already ran. Stories 557–622 and 679–690.

## The objects

| Object | Collection | What it is |
| --- | --- | --- |
| Envelope | `envelopes` | Caps (cost / month, cases / day, minutes / campaign, parallel jobs) plus gate rules, per org, project, role or user. A boost adds headroom until it expires. Every write is audited. |
| Campaign | `campaigns` | The costed plan: itemised work with rationale, sources, surface, locale, build; estimate with a confidence range; tranche A (runs now) and tranche B (needs a signature); immutable approved versions; results and estimate-vs-actual. |
| Campaign template | `campaign_templates` | A reusable item list. A standing approval on a template lets its gated tranche run under a cap without a new signature. |
| Approval request | `approval_requests` | A signature request with a role chain, deputies, escalation and expiry timers, flags streamed from tranche A, and every decision on it. |

Services: `apps/api/src/services/envelopes.ts`, `campaigns.ts`, `approvals.ts`, `preflight.ts`, `errors.ts`, `rehearsal.ts`.
Routes: `apps/api/src/routes/governance.ts`. Chat tools: `propose_campaign`, `campaign_status` (`agent/tools/campaign.ts`).

## Envelopes (557–566)

- The **org** sets the ceiling. A **project**, **role** or **user** envelope can only narrow it; an admin (`envelopes:manage`) may widen a project on purpose.
- The effective envelope for one person in one project is the **minimum of every layer**, boosts added to the layer that received them, minus this period's spend (`effectiveEnvelope`). The UI says where each cap came from ("your project cap is $40, the org allows $200").
- Rules only ever add gates: destructive work, security scans and messages outside the sandbox always ask; production targets always ask unless the org turns that off; a load item above the agreed virtual users or a model above the price threshold asks.
- The org envelope's `tenantQuotaConcurrency` is the whole-tenant ceiling on jobs running at once; the worker defers a job over the line instead of failing it.
- `GET /api/projects/:id/envelope` (mine, with headroom), `GET/POST/DELETE /api/orgs/:orgId/envelopes`, `POST …/envelopes/boost` (admins grant; anyone else's request becomes an approval), utilisation on the same GET.

## Campaign lifecycle (567–592)

1. **Propose.** The planner calls `propose_campaign` (or a person posts `POST /api/projects/:id/campaigns`). Every item is priced from the project's own finished runs — average cost and minutes per case, HITL rate, a range from the sample size — and gate reasons are computed against the effective envelope. Items are split: gated → B; the rest fill A cheapest-highest-signal first until the headroom is used, and what does not fit goes to B with the cap as the reason. Excluded items stay on the card with the reason.
2. **Edit.** `PATCH /api/campaigns/:id` re-prices live (exclude / include / replace items); every edit bumps the version. Only a *proposed* campaign is editable.
3. **Approve on the card.** `POST /api/campaigns/:id/decision` freezes the version into `approvedVersions`, sets the cap to the high estimate, enqueues tranche A as ordinary durable jobs (stamped with `campaignId / trancheId / campaignItemId`) and opens an approval request for tranche B. Reduce = approve with items excluded. Reject keeps the object for the record.
4. **Stream.** As tranche A finishes, its results are recomputed from the runs and the ledger (idempotent) and appear on the pending request as a flag ("1 defect found — review before approving more spend"). If half or more of A blocked or hit environment errors, B is withdrawn so nobody signs for a broken target.
5. **Sign.** Approved work enqueues itself; a partial approval runs only the chosen items; a rejection or expiry ends B while A's report stands; `POST /api/campaigns/:id/rerequest` asks again quoting the previous outcome. A late approval re-opens a campaign that finished on A alone.
6. **Finish.** When both tranches have ended, the campaign completes with the partial/complete note, estimate-vs-actual variance, and a Slack / Teams / push notice.

Controls while running: `pause`, `resume`, `halt` (cancels what is left, keeps everything completed), `throttle` (fewer parallel jobs and / or a cheaper step model), `cap` (raise the spend cap). Templates: `POST /api/campaigns/:id/template`, `POST /api/projects/:id/campaign-templates/:tid/campaigns`, standing approval on `…/standing` (manager or admin). Comparison: `GET /api/campaigns/:id/compare/:otherId`.

## Approvals (593–608)

- **Chain from the amount.** The org policy (`OrgSettings.approvals.chain`, default Lead → Manager above $100 → Admin above $1000) decides which levels must sign. Destructive work always adds a manager. Higher roles may sign lower levels; one person signs one level of a chain (four eyes).
- **Deputies** stand in for a named approver until their expiry (`POST /api/orgs/:orgId/approval-policy/deputy`).
- **Timers on the scheduler leader** (`sweepApprovals`, every minute): a quiet level is escalated to the next one and re-notified; a forgotten request expires and the in-envelope part of the campaign still stands.
- **Break-glass** needs `approvals:break_glass` (or a role the org named) and a written reason; it signs every remaining level at once and is audited as `approval:break_glass` and logged at warn level.
- **Batch** approval signs several non-destructive requests at once; destructive ones are skipped by name.
- **Self-approval is refused**, always, unless break-glass.
- **Notifications**: push + email to every eligible approver with signed one-tap links (`GET /api/approvals/:id/act?d=approve&u=…&exp=…&sig=…`, HMAC over request, approver, decision and expiry), one message to the project's Slack / Teams channel with the review link, and an `approval:request` event to every device of the approvers and the requester.
- **Pending panel**: `GET /api/approvals` returns what I must decide, what I am waiting on, batches I can sign together, and history. The web shows it on Runs (workspace) and on Mission control (managers / admins); `/approvals/<id>` deep links land on the request.

## Budget checkpoints (609–616)

The worker recomputes campaign spend after every run. At 50 % and 80 % of the cap a `budget:checkpoint` event and a push say how much is spent, how many cases are done and what finishing would cost. At 100 % every active job is asked to pause at its next safe boundary, the campaign is marked paused, and a `budget_checkpoint` approval opens: approve (optionally with a new cap) resumes from the exact checkpoint and re-arms the 100 % alert; reject halts and keeps the partial report. Throttle is available at any time. The runner-side guard (`ctx.budgetGuard`) only halts a case that runs 20 % past the cap, so a checkpoint is a pause between cases, not a torn run.

## Demo safety (679–690)

- **Preflight** (`POST /api/projects/:id/preflight`, Settings → Governance): model key and catalog, provider, browser pool pressure, target reachability and login material, connections, desktop workers, envelope headroom, platform. Each check has a state and a fix.
- **Error catalog** (`services/errors.ts`): every run, job and notice gets a sentence with a next step instead of a raw message; provider outages name the alternative model and say the checkpoint is kept.
- **Stall self-report**: a tool call still running after six seconds says so in the status line, and keeps saying so.
- **Rehearsal mode** (`POST /api/projects/:id/rehearsal`): pins models, skills, MCP servers, disabled tools and run settings; settings writes that would change them are refused with 409 until it is off (`force=true` overrides); the pinned model is used even if the catalog would swap it.
- **Degraded mode**: `run.limitations[]` lists what a run could not deliver (video, provider degraded, …) and the report shows it up front.
- **Platform health**: `/api/health.platform` and the panel in Settings → Governance.

## Honest limitations

- Slack / Teams receive review links, not interactive buttons (a Slack app is needed for buttons).
- Estimates come from the project's own history; a new project prices at Flash-class defaults and says so in the basis line.
- The budget guard sees the spend of its own run plus the campaign's spend at job start; concurrent jobs are reconciled between cases, so the hard stop is the 100 % pause, not the per-tick guard.
