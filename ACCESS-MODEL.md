# Access model — testing protected environments (stories 836–845)

The rule in one line: **we never evade protection; we ask for a lane through it.** When a target refuses an
automated browser, the platform names the mechanism, records the evidence, marks the case `BLOCKED (access)` —
not a product defect — and lists the remedies a security team can grant. Nothing in the codebase solves CAPTCHAs,
rotates residential proxies, forges fingerprints or replays attestation tokens, and nothing will.

## 1. What we detect

`classifyBlock()` in `packages/browser-driver` reads the final URL, title, body text, frames, cookie names, HTTP
status and response headers of the main document and returns a `BlockReport`:

| mechanism | typical evidence | default remedies |
|---|---|---|
| `cloudflare` | "Just a moment…", `cf-mitigated: challenge`, `__cf_bm`, challenge frame | allowlist egress IPs · WAF skip rule for the test-traffic header · saved login |
| `akamai` | `_abck` / `bm_sz` cookies, "Access Denied" reference number | Bot Manager allowlist for egress IPs or a test-traffic header |
| `perimeterx` | `_px*` cookies, `px-captcha` frame | HUMAN allowlist rule for the header / IPs |
| `datadome` | `datadome` cookie, `x-datadome` header, geo.captcha-delivery.com frame | DataDome allowlist rule |
| `imperva` | `visid_incap_*` / `incap_ses_*`, Incapsula incident id | Imperva allowlist / test-traffic rule |
| `captcha` | reCAPTCHA / hCaptcha / Turnstile frame with no other vendor | test key or disabled challenge for the QA environment · a human solves it via take control |
| `waf` | 403 with WAF vocabulary ("request blocked", "forbidden by policy") | egress allowlist |
| `attestation` | Play Integrity / App Attest / device-check wording | QA build with attestation off or a test attestation key |
| `rate_limit` | 429, `retry-after`, "too many requests" | raised limits for our egress or a lower run concurrency |
| `login_wall` | password field with few other actions | saved login (`inject_auth_state`) or credentials via `request_user_help` |
| `unknown` | nothing matched but the page is empty / refuses | check the URL, VPN and allowlist |

Detection runs on demand (`check_blockers`, `preflight_access`) and whenever a run ends: the runner stamps the
report on `run.block` and the verdict becomes `BLOCKED`. The web run page shows it as the **Block card** with the
evidence and the remedies; there is no "retry harder" button.

## 2. The lanes we support

1. **Egress allowlist (840).** `GET /api/health/egress` (also the `egress_addresses` tool) reports the public
   addresses our traffic comes from: the API host, the Browserless pool (probed from inside a live browser so it
   is the address the WAF sees), and each paired desktop worker's last reported address. Cloud egress can change
   on redeploy — the note says so and `preflight_access` re-checks before a campaign.
2. **Test-traffic headers (839).** A vault entry of kind `bypass_header` holds header name/values scoped to a
   host pattern (`*.staging.example.com`) and optionally an environment. `bypassHeadersFor()` resolves them when
   a browser launches; the driver adds them to requests to matching hosts only — never to third parties — and
   records a `header` manipulation so the run says the lane was used. Values are encrypted like every vault
   secret and never shown in the UI or in tool output.
3. **Saved logins (841–842).** When a human takes control of the live browser, signs in and hands control back,
   the cookies and origin storage for that host are saved to the auth-session registry (`auth_sessions`,
   encrypted, 7-day expiry by default). `inject_auth_state` reuses them; `save_auth_session` saves the agent's own
   login; `list_auth_sessions` shows what exists. An expired session is reported as expired with the refresh path,
   never silently used.
4. **Environment tweaks the customer owns.** CAPTCHA test keys, attestation off for the QA build, a lower rate
   limit threshold for the allowlisted addresses. We list them on the card; the customer applies them.
5. **A human in the loop.** Take control from the live pane, solve the challenge or sign in, hand back. The
   hand-back saves the session (lane 3), so it happens once per expiry period, not once per run.

## 3. Preflight (844)

`preflight_access url` opens the target, classifies any protection, checks the registry for a saved login and
the vault for bypass headers, and reports egress addresses. It is read-only and cheap; the lead runs it before a
campaign on a new environment so the approval card is not spent on `BLOCKED` results.

## 4. What is recorded

- `run.block` — mechanism, evidence, remedies, URL, status, time.
- `run.manipulations` — every header lane and auth injection applied (kind `header` / `auth`), shown under the
  run's **Changes** tab.
- `auth_sessions` — host, role, label, counts, who saved it, when, expiry, source (`take_control` / `agent` /
  `import`). The cookies themselves are encrypted at rest and only decrypted into a browser context.
- Audit — vault writes and approvals follow the governance audit trail (see `docs/CAMPAIGNS.md`).

## 5. Limitations (stated, not hidden)

- We do not solve CAPTCHAs, defeat bot managers or spoof devices. A `captcha` block without a test key needs a human.
- Egress addresses of a Railway service are not fixed; re-check before allowlisting for a campaign.
- Bypass headers are a lane the customer grants; the platform cannot mint them.
- Attestation on mobile is detected from wording only; the fix is a QA build, which the customer produces.
- Browserless Chromium advertises a desktop Chrome user agent to keep naive filters quiet; real Firefox / WebKit
  engines keep their own UA so an engine-specific bug is reproduced truthfully (816).
