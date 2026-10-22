# Non-Functional Testing — User Stories

*Continues the backlog. Stories 306–357. Covers load, stress, soak, spike, scalability, security/VAPT, resilience, and non-functional reporting.*

---

## The reframe

Your agent handed you a k6 script instead of running the test. The fix isn't only "stop refusing." It's three things:

1. **The agent owns execution.** The deliverable is an executed run and a report, inside the product. No copy-paste into a terminal.
2. **It commands a load substrate, it isn't one.** A browser-driving agent can't itself be 1,000 users. The feature is: the agent generates a load definition, provisions runners capable of real concurrency, executes, streams metrics, and writes the report. Without this, any "1,000 users" claim is fiction.
3. **Authorization replaces refusal.** Verify the target is yours once, then let high-volume runs proceed — bounded by hard caps and an instant kill switch so an accident can't take down the target. That's how the product says yes safely, which is exactly what makes it sellable rather than dangerous.

Everything below assumes those three.

---

## Y. Owning execution — the agent runs it, not you

**306. [P0]** As a tester, I want the agent to run the load test and return results, not hand me a script, so that the tool does the work.
`Pass:` A load request produces an executed run with metrics and a report inside the product, with no terminal step.

**307. [P0]** As a tester, I want the load scenario generated from a journey I describe or already tested, so that I don't author load scripts.
`Pass:` "Log in, open the dashboard, run a search" becomes a runnable scenario with those steps and their data parameterised.

**308. [P0]** As a tester, I want the target concurrency actually generated, so that "1,000 users" means 1,000, not one browser looping.
`Pass:` A 1,000-VU run shows ~1,000 concurrent sessions in the metrics, produced by a load-generation substrate rather than the agent's own session.

**309. [P1]** As a tester, I want to choose protocol-level or browser-level load, so that I trade realism against cost.
`Pass:` Protocol-level (raw HTTP) runs cheaply at high VU counts; browser-level runs a real rendered page at lower counts; the agent explains the tradeoff and defaults sensibly.

**310. [P1]** As a tester, I want authentication handled correctly in the load run, so that logged-in flows work at scale.
`Pass:` A token or session is acquired per virtual user (or reused, per config); the run doesn't collapse to a single shared session.

**311. [P1]** As a tester, I want dynamic response values captured and reused within each user's flow, so that multi-step scenarios survive at scale.
`Pass:` A value returned in step 1 (order ID, CSRF token) is extracted and used in step 3 without me writing the correlation.

**312. [P1]** As a tester, I want each virtual user to draw from a data set, so that I'm not hammering one account.
`Pass:` An uploaded account list is distributed across VUs; the report shows no single-account contention artefacts.

**313. [P1]** As a tester, I want to shape the load profile in plain language, so that I model real traffic rather than a flat line.
`Pass:` "Ramp to 1,000 over two minutes, hold five minutes, spike to 2,000 for thirty seconds, ramp down" executes as described.

**314. [P0]** As a tester, I want to watch metrics stream during the run, so that I can stop early if something's clearly wrong.
`Pass:` Active VUs, RPS, p50/p95/p99 latency, and error rate update live; a stop control ends the run cleanly.

**315. [P2]** As a tester, I want load generated from multiple regions, so that latency and CDN behaviour are realistic.
`Pass:` The run distributes across configured regions and reports latency per region.

**316. [P0]** As a tester, I want a readable report at the end, not raw numbers, so that I can act and share.
`Pass:` Verdict against thresholds, percentile latency, error breakdown by status and type, throughput, and the load level at which degradation began.

**317. [P1]** As a tester, I want the agent to interpret the result, so that a non-specialist knows what to do next.
`Pass:` "Errors begin around 340 VUs, all on /search, 503s consistent with database connection-pool exhaustion — raise the pool or add caching" — not just a chart.

---

## Z. The load-testing family

**318. [P0]** As a tester, I want a standard load test at expected peak, so that I can confirm we meet SLOs under normal busy conditions.
`Pass:` Run at target concurrency reports pass/fail against latency and error SLOs.

**319. [P0]** As a tester, I want a stress test that finds the breaking point, so that I know the ceiling before launch does.
`Pass:` Agent ramps until failure and reports the load at which errors began and how it failed — slow, erroring, or silently wrong.

**320. [P1]** As a tester, I want a spike test, so that I know what a sudden surge does.
`Pass:` A sharp jump is applied; the report says whether the system absorbed it, shed load gracefully, errored, or fell over, and how long recovery took.

**321. [P1]** As a tester, I want a soak/endurance test over hours, so that leaks and slow degradation surface.
`Pass:` Sustained load for a long window reports memory growth, response-time drift, and connection exhaustion as trends.

**322. [P1]** As a tester, I want a scalability step test, so that I can see where linear scaling breaks.
`Pass:` Load increases in steps; a throughput-vs-load curve is produced with the knee (the point returns stop scaling) identified.

**323. [P2]** As a tester, I want volume testing with large data and payloads, so that size-related failures are caught.
`Pass:` Behaviour with large result sets, big uploads, and heavy pagination is reported.

**324. [P2]** As a tester, I want a capacity estimate against a target, so that I can plan infrastructure.
`Pass:` "Current setup sustains ~600 concurrent within SLO; reaching 1,000 needs roughly X" with the limiting resource named.

**325. [P1]** As a tester, I want a quick single-endpoint hammer, so that I can test one thing fast — like a login.
`Pass:` A named endpoint is driven at a chosen rate and its latency and error profile reported, without building a full journey.

**326. [P1]** As a tester, I want this run compared to a saved baseline, so that performance regressions are attributed to the change that caused them.
`Pass:` "Login p95 is 1.8s slower than last week's run" against a stored baseline.

---

## AA. Authorization and blast radius — how the product says yes safely

**327. [P0]** As a tester, I want to verify I own a target once, so that high-volume runs against it are then allowed.
`Pass:` Ownership is confirmed via a DNS token, a response header, or a workspace-verified domain; afterwards, load and security runs proceed without per-run friction.

**328. [P0]** As a tester, I want unverified external targets blocked with a path to verify, so that the tool can't be pointed at someone else's site.
`Pass:` An unverified domain returns a "verify ownership to proceed" flow — not a flat refusal and not silent execution.

**329. [P0]** As a tester, I want hard caps and an instant kill switch, so that a run can never spiral.
`Pass:` Per-project maximums on VUs, request rate, and duration are enforced; a running test can be aborted in one action and stops generating traffic within seconds.

**330. [P1]** As a tester, I want production targets to demand extra confirmation and off-peak scheduling, so that I don't take down live infrastructure.
`Pass:` A prod target requires explicit acknowledgement and can be scheduled into a low-traffic window.

**331. [P1]** As a tester, I want the agent to auto-abort if the target itself is in distress, so that the test protects the system it's testing.
`Pass:` The agent watches a health signal and halts if target error rate or latency crosses a safety line, reporting why it stopped.

**332. [P1]** As a tester, I want a smoke run before a full run, so that mistakes are cheap.
`Pass:` The agent proposes a small warm-up (e.g. 50 VUs) before a 1,000-VU run, and validates the scenario works before scaling it.

**333. [P1]** As a QA lead, I want load spend capped, so that a runaway test doesn't run up a bill.
`Pass:` A per-run and per-project budget halts execution at the limit and returns a partial report.

---

## AB. Security and VAPT basics

> Scope: enough coverage that basic issues surface continuously, not a replacement for a full penetration test. Findings are proven, never weaponised.

**334. [P0]** As a tester, I want both authenticated and unauthenticated scans, so that I see what an outsider and a logged-in user can each reach.
`Pass:` Both modes run; the report separates findings by which access level exposed them.

**335. [P1]** As a tester, I want authentication security checked, so that the login itself isn't the weak point.
`Pass:` Reports on brute-force protection and lockout, weak-password policy, session fixation, tokens exposed in URLs, and whether logout truly invalidates the session.

**336. [P1]** As a tester, I want access-control and IDOR checks, so that users can't reach each other's data.
`Pass:` Using two role accounts, the agent alters record identifiers and forces URLs, then reports whether horizontal and vertical access control held.

**337. [P1]** As a tester, I want injection surfaces probed, so that unsanitised inputs are caught.
`Pass:` SQL, XSS (reflected and stored), and command/template payloads are tested on inputs; the report names where input was reflected or executed, with a minimal proof and no further exploitation.

**338. [P1]** As a tester, I want transport and configuration hygiene checked, so that the basics aren't missing.
`Pass:` A per-item result for TLS version and ciphers, HSTS, security headers (CSP, X-Frame-Options, and peers), cookie flags (HttpOnly, Secure, SameSite), and CORS configuration.

**339. [P1]** As a tester, I want information-exposure checks, so that we don't leak our own internals.
`Pass:` Reports API keys embedded in client JS, verbose errors and stack traces, directory listing, and source maps served in production.

**340. [P1]** As a tester, I want rate-limiting and abuse resistance checked, so that endpoints like login can't be pounded freely.
`Pass:` Repeated automated requests are made against sensitive endpoints; the report says whether throttling, backoff, or blocking kicked in.

**341. [P2]** As a tester, I want findings mapped to a recognised taxonomy, so that they're legible to security teams.
`Pass:` Each finding is tagged to an OWASP Top 10 category (or similar) with severity.

**342. [P2]** As a tester, I want exposed components fingerprinted against known vulnerabilities, so that outdated dependencies surface.
`Pass:` Server, framework, and visible library versions are identified and flagged where known CVEs exist.

**343. [P0]** As a security reviewer, I want scans confined to the authorised scope, so that we never scan something we don't own.
`Pass:` The scan stays within the verified domain and declared scope; out-of-scope hosts encountered (third-party embeds, redirects) are skipped and noted.

**344. [P0]** As a security reviewer, I want findings proven, not exploited, so that testing never causes harm or exfiltration.
`Pass:` Each finding carries a minimal proof of concept; no data is dumped, no records altered, no destructive action taken.

**345. [P1]** As a tester, I want each finding triaged and deduplicated, so that the report is actionable.
`Pass:` Severity, evidence, and remediation guidance per finding; findings sharing a root cause are grouped.

**346. [P2]** As a tester, I want to re-test a prior finding, so that I can confirm a fix.
`Pass:` Re-running a finding returns a closed/reopened status with fresh evidence.

---

## AC. Resilience and reliability

**347. [P1]** As a tester, I want dependency failure injected, so that failure behaviour is designed rather than discovered in production.
`Pass:` The agent forces a downstream 500 or timeout and reports whether the app degraded gracefully, surfaced a clear error, corrupted state, or lost work.

**348. [P1]** As a tester, I want behaviour under network degradation tested, so that flaky-connection users are accounted for.
`Pass:` Latency, throttling, and packet loss are applied; the report describes spinner, timeout, retry, and data-integrity behaviour.

**349. [P2]** As a tester, I want failover and recovery observed, so that I know how the system heals.
`Pass:` Where a dependency can be disrupted or simulated, the agent reports recovery time and whether data survived.

**350. [P2]** As a tester, I want overload/backpressure behaviour tested, so that I know if the system sheds load or collapses.
`Pass:` Under sustained overload, the report says whether requests were queued, rejected cleanly (429), or failed uncontrollably.

---

## AD. Reporting, SLOs, and closing the loop

**351. [P0]** As a tester, I want one unified non-functional report per target, so that performance, security, and resilience results aren't scattered.
`Pass:` A single report aggregates the run types executed against a target with an overall readiness verdict.

**352. [P1]** As a tester, I want to define SLOs once and reuse them as a gate, so that pass/fail is objective and repeatable.
`Pass:` Saved thresholds (latency, error rate, finding severity) drive run verdicts and can gate a CI pipeline.

**353. [P1]** As a tester, I want client metrics correlated with server-side data, so that a spike points at a cause.
`Pass:` A latency spike links to the corresponding server trace, log window, or APM metric where a connection exists.

**354. [P1]** As a tester, I want non-functional results trended over time, so that slow drift is visible.
`Pass:` p95, error rate, and finding counts are tracked across runs with regressions surfaced.

**355. [P1]** As a tester, I want to promote a finding into a permanent check, so that results become guards.
`Pass:` One action turns a load SLO or a security finding into a recurring, scheduled check.

**356. [P2]** As a tester, I want scheduled non-functional runs with alerting, so that regressions are caught without me initiating them.
`Pass:` Nightly or weekly load and security runs execute and alert only when something crosses a threshold.

**357. [P1]** As a tester, I want raw results exportable, so that our specialists can dig in with their own tools.
`Pass:` Underlying data exports in a standard form (k6/JMeter-compatible or JSON), alongside the shareable summary.

---

## Build order

**Prove real concurrency first (306, 308, 316).** Until the agent can actually generate 1,000 users on a substrate and hand back a report, nothing else in this section is real. This is infrastructure, and it's the hard part — do it before the breadth.

**Then the authorization spine (327–329).** Ownership verification, caps, and a kill switch are what let you ship any of this to real users. They're small and they're the difference between a feature and a liability.

**Then the load family (318–322).** Load, stress, spike, soak, scalability reuse the same substrate — mostly a matter of load profile and reporting. High value for low marginal cost once the engine exists.

**Security (334–345) is a parallel track.** Different engine, different expertise. Ship the transport/config/headers checks (338) and rate-limiting (340) first — they're deterministic, low-false-positive, and directly answer the login example you started with. Injection and IDOR are higher value but need more care to keep false positives down.

**One honest scoping note.** "Basic VAPT" that you can stand behind is worth far more than broad security claims you can't. Under-promise the security surface and be precise about what it does and doesn't cover — a security team that catches your tool over-claiming will distrust everything else it says.
