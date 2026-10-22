# Authorized, Ownership-Verified Security-Hygiene & Performance Testing — User Stories

*Continues the backlog. Stories 500–524. Companion to the non-functional stories (306–357), the ownership-verification stories (358–375), the enterprise layer (376–438) and the UI stories (439–487). Renumbered from 376–400 to avoid colliding with the enterprise backlog.*

Every story here frames the work as **confirming a safeguard is in place and reporting findings with remediation** — not as performing an attack. The agent only runs these against an origin the project has verified it controls (see stories 358–375); it proves findings with minimal evidence, dumps no data, alters no records, and stays inside the declared scope. Each story says what protection is verified and what the report must contain.

---

## AI. The authorization contract (what every story below assumes)

**500. [P0]** As a security reviewer, I want each of these checks to run only against a verified origin and declared scope, so that testing is always authorized.
`Pass:` The check refuses an unverified or out-of-scope host and returns the verification path (story 372); a run records which origin, scope, and environment it was authorized against.

**501. [P0]** As a security reviewer, I want every finding proven, never exploited, so that the check can never cause harm.
`Pass:` Each finding carries a minimal, non-destructive proof (a status code, a header value, a boundary result) with no data exfiltration and no state change; the report says so explicitly.

**502. [P1]** As a QA lead, I want a single readiness verdict per target, so that "is this safe to ship" has one answer.
`Pass:` A unified report aggregates the safeguard checks below with an overall pass/attention/fail verdict and a count of findings by severity.

---

## AJ. Rate-limiting & abuse-resistance enforcement

**503. [P0]** As a tester, I want to confirm sensitive endpoints (login, password reset, OTP) enforce rate limiting, so that they can't be pounded freely.
`Pass:` A bounded burst of authorized requests is sent; the report states whether throttling, backoff (429/503 + Retry-After), or lockout engaged and after how many attempts. If none did, it is a finding.
`Report:` endpoint, attempts sent, status-code distribution, first-limited-at count, Retry-After value, verdict (safeguard present / absent), remediation (per-IP and per-account limits, progressive lockout).

**504. [P1]** As a tester, I want to confirm progressive lockout on repeated failed logins, so that credential-stuffing is slowed.
`Pass:` Repeated deliberately-wrong logins (bounded, using synthetic credentials) are made; the report says whether lockout or increasing backoff kicked in, and after how many attempts, without ever locking a real user's account it shouldn't.
`Report:` attempts, lockout threshold observed, backoff pattern, account used (synthetic), remediation.

**505. [P2]** As a tester, I want abuse resistance confirmed on unauthenticated write endpoints, so that public forms aren't a spam vector.
`Pass:` A bounded burst against a public POST endpoint reports whether throttling/CAPTCHA/quota engaged.
`Report:` endpoint, throttle behaviour, verdict, remediation.

---

## AK. Access-control boundaries between roles

**506. [P0]** As a tester, I want horizontal access-control confirmed between two same-role accounts, so that one user can't read another's data.
`Pass:` With two verified accounts, the agent requests account B's record as account A (altering an identifier) and reports whether the boundary held (403/404) or leaked (2xx with B's data).
`Report:` endpoint, both roles, status per role, body-equality result, verdict (boundary held / IDOR finding), remediation (enforce object-level authorization server-side).

**507. [P0]** As a tester, I want vertical access-control confirmed between roles, so that a lower-privilege role can't reach admin functions.
`Pass:` A viewer/low-privilege token is used against a privileged action; the report says whether it was correctly denied.
`Report:` action, roles compared, status codes, verdict, remediation.

**508. [P1]** As a tester, I want forced-browsing of privileged URLs confirmed as blocked, so that hidden pages aren't reachable by guessing.
`Pass:` Known privileged paths are requested with a low-privilege session; the report says whether they were denied or exposed.
`Report:` paths tried, status per path, verdict, remediation.

**509. [P1]** As a security reviewer, I want authenticated and unauthenticated views separated, so that I see what an outsider vs a logged-in user can each reach.
`Pass:` The same checks run in both modes; findings are labelled by the access level that exposed them.
`Report:` per-finding access-level tag.

---

## AL. Transport, header & cookie configuration

**510. [P0]** As a tester, I want transport security confirmed, so that the connection itself isn't the weak point.
`Pass:` Per-item results for TLS version (legacy 1.0/1.1 accepted?), HSTS present, and HTTPS-only.
`Report:` each item as safeguard/status/remediation; TLS ≥ 1.2 required, HSTS with a sensible max-age.

**511. [P0]** As a tester, I want security headers confirmed present, so that common client-side attacks are limited.
`Pass:` CSP, X-Frame-Options (or frame-ancestors), X-Content-Type-Options=nosniff, Referrer-Policy each reported present or missing.
`Report:` checklist with the exact header to add for each miss.

**512. [P1]** As a tester, I want cookie flags confirmed, so that session cookies resist theft and CSRF.
`Pass:` Session/auth cookies reported for Secure, HttpOnly, and SameSite.
`Report:` per-cookie flag matrix, verdict, remediation (Secure + HttpOnly + SameSite=Lax/Strict on session cookies).

**513. [P1]** As a tester, I want CORS configuration confirmed safe, so that other sites can't read authenticated responses.
`Pass:` The report says whether the origin reflects an arbitrary Origin, and whether it combines a wildcard with credentials.
`Report:` observed ACAO/ACAC behaviour, verdict, remediation (allow only known origins; never `*` with credentials).

---

## AM. Information-exposure checks

**514. [P1]** As a tester, I want confirmation that secrets aren't shipped to the client, so that keys aren't harvestable from JS.
`Pass:` Client bundles/network are scanned for API keys/tokens; the report names any exposure (redacted) and its location.
`Report:` file/URL, kind of secret (redacted), verdict, remediation (move to server, rotate).

**515. [P1]** As a tester, I want confirmation that errors don't leak internals, so that stack traces and versions aren't public.
`Pass:` Error responses are checked for stack traces, framework/version banners, and verbose messages.
`Report:` endpoint, what leaked, remediation (generic error pages, strip Server/X-Powered-By).

**516. [P2]** As a tester, I want confirmation that source maps and directory listings aren't served in production, so that source and structure aren't exposed.
`Pass:` The report says whether `.map` files, directory listings, or common sensitive paths are reachable.
`Report:` paths, status, verdict, remediation.

---

## AN. Capacity limits under authorized load

**517. [P0]** As a tester, I want the sustainable capacity within SLO confirmed, so that I know the safe operating ceiling.
`Pass:` An authorized load run reports the highest concurrency held within the latency and error-rate SLOs.
`Report:` peak VUs, sustained-capacity VUs, p50/p95/p99, error rate, verdict vs SLO.

**518. [P0]** As a tester, I want the degradation point identified, so that I know where and how it first breaks.
`Pass:` A ramp reports the concurrency at which latency/errors first breach SLO and the failure mode (5xx/timeouts/resets).
`Report:` degradation-at-VUs, dominant failure status, interpretation, remediation.

**519. [P1]** As a tester, I want surge recovery confirmed, so that a spike doesn't cause lasting damage.
`Pass:` A spike run reports whether the system absorbed, shed cleanly (429), errored, or fell over, and how long it took to recover.
`Report:` spike level, behaviour, recovery time, verdict.

**520. [P1]** As a tester, I want stability over time confirmed, so that leaks and drift surface before production does.
`Pass:` A soak run reports latency drift and error-rate trend over a sustained window.
`Report:` window, p95 drift, error trend, verdict, remediation if drift is non-flat.

**521. [P1]** As a tester, I want load runs kept within safe bounds, so that a capacity check can never become an outage.
`Pass:` Hard caps (VUs, duration, requests), a smoke run first, an instant kill switch, and distress auto-abort are all enforced and reflected in the report when they fire.
`Report:` caps applied, whether smoke passed, whether the run auto-aborted and why.

---

## AO. Findings lifecycle

**522. [P1]** As a tester, I want each safeguard finding filed as a tracked issue, so that it's actionable and doesn't get lost.
`Pass:` Findings become issues with severity, evidence, and remediation; duplicates sharing a root cause are grouped.

**523. [P1]** As a tester, I want to re-test a finding after a fix, so that closure is evidence-based.
`Pass:` Re-running a finding returns closed/reopened with fresh evidence against the same origin and scope.

**524. [P2]** As a tester, I want a safeguard check promotable to a recurring guard, so that a confirmed protection stays confirmed.
`Pass:` One action turns a hygiene/rate-limit/capacity check into a scheduled run that alerts only on regression.

---

## Build order

1. **500–501 (the contract) + 510–513 (transport/header/cookie/CORS) + 503 (rate limiting).** Deterministic, low-false-positive, and they directly answer the login example. Shipped first via `security_hygiene`, `rate_limit_probe`, and the ownership gate.
2. **506–509 (access-control boundaries).** Higher value, needs two verified role accounts and care to avoid false positives; built on `idor_check`.
3. **517–521 (capacity under load).** Reuses the load substrate; mostly profile + reporting.
4. **514–516 (information exposure) and 522–524 (lifecycle).** Valuable, incremental; information-exposure needs the browser + network capture, lifecycle leans on the existing issue tracker and scheduler.

**Scoping note.** Under-promise the security surface. This is basic, authorized, defensive hygiene — safeguards confirmed and findings reported with remediation — not a substitute for a full penetration test. A precise claim you can stand behind is worth more than a broad one you can't.
