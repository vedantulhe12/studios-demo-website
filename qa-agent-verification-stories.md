# Ownership Verification & Authorized Testing — User Stories

*Continues the backlog. Stories 358–375. These harden the "Confirm you own this app" flow so it works reliably and safely, and so an agent never stalls on something it isn't allowed to do.*

---

## The principle

The flow you've built is the right shape: **prove control of the origin once, then let the risky test types run.** The value of these stories is in the states the happy-path modal doesn't cover — expiry, revocation, partial trust, and the moment the *agent* (not the user) hits the wall mid-run. That last one is what turned your k6 request into a dead end, and it's the one worth getting right.

Two ideas run through all of them:

- **Graduated trust, not a single gate.** Read-only exploration needs no proof. Load and security testing need origin control. Production-scale runs might need more. One binary "verified / not verified" flag collapses distinctions that matter.
- **The agent proposes, the gate authorizes.** When the agent wants to do something it isn't cleared for, it should surface the exact unlock — not refuse, and not silently skip.

---

## AE. Verification that holds up

**358. [P0]** As a tester, I want to start functional QA immediately and only hit verification when I request a test type that needs it, so that the gate never blocks work that doesn't require it.
`Pass:` Read-only and functional testing run with no verification; the ownership modal appears only when load, stress, or security is requested. (Your modal already says this — the story is making sure the tiers are actually enforced, not just described.)

**359. [P0]** As a tester, I want the "Check now" result to tell me exactly why it failed when it fails, so that I can fix it without guessing.
`Pass:` A failed check distinguishes: file not found (404), returned a redirect, returned the wrong value, returned extra characters or wrapping HTML/JSON, wrong content-type, or blocked by auth — each with the specific fix.

**360. [P1]** As a tester, I want the check to tolerate trivial formatting differences, so that a stray newline doesn't cost me twenty minutes.
`Pass:` Leading/trailing whitespace and a trailing newline are accepted; the value match is otherwise exact, and the message says what it normalized.

**361. [P1]** As a tester, I want to retry the check without restarting the whole flow, so that deploy-and-recheck is fast.
`Pass:` "Check now" is re-runnable in place; the nonce stays stable across retries so I don't have to re-serve a new value each time.

**362. [P1]** As a tester, I want to verify via more than one method, so that I'm not blocked when I can't serve a static file.
`Pass:` At least one alternative to the `.well-known` file exists — a DNS TXT record or a response header — and the flow lets me pick.

**363. [P2]** As a tester, I want the nonce to expire and be rotatable, so that a leaked verification value can't be reused indefinitely.
`Pass:` An unused nonce expires after a stated window; I can regenerate it; the old value stops working immediately.

---

## AF. Scope — what "owning the app" actually authorizes

**364. [P0]** As a tester, I want verification scoped to a specific origin, so that proving one domain doesn't silently authorize a different one.
`Pass:` Verifying `kadeep.ai` does not authorize `api.kadeep.ai`, `staging.kadeep.ai`, or `kadeep.ai:8443` without their own check; subdomain coverage is explicit, never assumed.

**365. [P1]** As a tester, I want to declare which paths and environments are in scope, so that testing stays inside what I intended.
`Pass:` An allowed-path list and environment tag are part of the authorization; requests outside them are refused and reported, not quietly attempted.

**366. [P1]** As a tester, I want third-party origins encountered mid-test skipped by default, so that I never send traffic at something I don't own.
`Pass:` When a flow calls out to a payment provider, CDN, or auth SaaS, the agent skips load/security actions against it, notes it, and continues against my origin.

**367. [P1]** As a tester, I want production explicitly distinguished from staging in the authorization, so that a prod run is always a deliberate act.
`Pass:` The environment is labeled on the authorization; a run against a prod-tagged origin requires a separate, explicit confirmation each time.

---

## AG. Revocation and lifecycle

**368. [P0]** As a project owner, I want to revoke verification at any time, so that authorization isn't permanent.
`Pass:` Revoking immediately disables the test types that required it; in-flight runs stop; re-verification is required to resume.

**369. [P1]** As a tester, I want the product to re-confirm the file is still live before a high-impact run, so that a domain that changed hands can't still be hammered.
`Pass:` Before a large load or security run, the nonce is re-checked; if it's gone, the run is held and I'm prompted to re-verify.

**370. [P1]** As a project owner, I want every verification, run, and revocation logged, so that there's an auditable record of who authorized what.
`Pass:` Each event records identity, origin, method, timestamp, and outcome, exportable for review.

**371. [P1]** As a QA lead, I want verification to belong to the project, not the person, so that a teammate isn't re-verifying an already-trusted app.
`Pass:` Once an origin is verified in a project, authorized members inherit it; verification survives the verifier leaving the team.

---

## AH. The agent at the wall — never a dead end

> This is the section that fixes your original experience. When the agent wants to do something it isn't cleared for, it should hand you the unlock, not the exit.

**372. [P0]** As a tester, I want the agent to route me straight into verification when a request needs it, so that I'm never told "I can't" with no way forward.
`Pass:` Asking for a load or security test on an unverified origin opens the ownership flow inline and resumes the original request automatically once the check passes — no re-asking.

**373. [P0]** As a tester, I want the agent to name the specific missing authorization, so that I know exactly what to unlock.
`Pass:` "This needs origin ownership for `kadeep.ai` and a raised VU cap" — not a generic refusal, and not a handoff to an external tool.

**374. [P1]** As a tester, I want the agent to offer the safe subset it *can* do right now, so that I get value while verification is pending.
`Pass:` While ownership is unconfirmed, the agent offers the functional, accessibility, and validation testing that needs no proof, and clearly marks what's waiting on the gate.

**375. [P1]** As a tester, I want a pending request to resume itself once I clear the gate, so that clearing verification feels like continuing, not restarting.
`Pass:` After "Check now" passes, the held load or security run starts with its original parameters intact, without me re-entering them.

---

## Build order

**372 and 375 are the ones to add now, while the agent is building.** They're the difference between your original experience — a refusal and a k6 script — and the experience the screenshot is reaching for: a wall that opens into a door and then puts you back where you were. Everything else hardens the gate; these two make it feel guided instead of blocked.

**359 is the cheap reliability win.** Most verification failures are boring — a redirect, a wrapped value, a trailing newline. Naming the exact reason turns a frustrating loop into a ten-second fix, and it's the difference between users completing verification and abandoning it.

**364 is the safety-critical one.** Scoping trust to a single origin is what stops "I verified my marketing site" from quietly authorizing an attack against an API on a neighboring subdomain. Get this boundary exactly right before you expand what verification unlocks.
