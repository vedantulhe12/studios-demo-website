# Integrations & Video Evidence — User Stories

*Continues the backlog. Stories 504–556.*

**Scope for v1:** email, SMS, WhatsApp as *receiving* channels (verifying what the app sent), plus Slack/Teams as *outbound* (delivering reports). Everything else is later.

**The core primitive: a test identity.** The agent needs an email address and a phone number it owns and can read. It enters those into the app under test, then checks what arrived. Without this, notification testing is impossible no matter how many connectors exist.

---

## BF. Test identities — the inbox the agent owns

**504. [P0]** As a tester, I want the agent to have an email address it can receive on, so that it can verify what my app actually sent.
`Pass:` The project has a working test address; the agent can enter it into a form and then read the message that arrives.

**505. [P0]** As a tester, I want a unique address per run, so that messages are attributable and runs don't collide.
`Pass:` Each run gets a distinct address (subaddressing or unique mailbox); a message is matched to the run that caused it, never to a parallel run.

**506. [P0]** As a tester, I want a phone number the agent can receive SMS on, so that OTP and alert flows are testable.
`Pass:` A test number is available; the agent reads an inbound SMS and extracts its content.

**507. [P1]** As a tester, I want a WhatsApp destination the agent can receive on, so that WhatsApp notifications are verifiable.
`Pass:` The agent reads an inbound WhatsApp message, including its template content where applicable.

**508. [P1]** As a tester, I want to use my own mailbox or number instead of a provided one, so that I can test with real infrastructure.
`Pass:` I can connect my own IMAP mailbox or messaging provider and the agent reads from it with the same behaviour.

**509. [P1]** As a tester, I want the inbox cleared between runs, so that an old message never satisfies a new assertion.
`Pass:` Each run starts from an empty state; a message from a prior run cannot be matched as a fresh one.

**510. [P1]** As a tester, I want to see the raw message the agent received, so that I can debug when an assertion fails.
`Pass:` The full message — headers, body, HTML, attachments — is viewable in the run evidence.

---

## BG. Verifying what arrived

**511. [P0]** As a tester, I want to assert that a notification arrived, so that I can prove the flow works end to end.
`Pass:` "Sign up, then confirm the welcome email arrives" passes when it does and fails clearly when it doesn't.

**512. [P0]** As a tester, I want to wait for an async message with a timeout, so that delivery latency doesn't cause false failures.
`Pass:` The agent polls until the message arrives or the timeout expires; the report states how long it waited and how long delivery took.

**513. [P0]** As a tester, I want assertions on message content, so that I catch wrong or broken messages, not just missing ones.
`Pass:` Sender, subject, body text, and links can each be asserted; a mismatch reports expected vs actual.

**514. [P1]** As a tester, I want unresolved template placeholders flagged automatically, so that broken personalisation never ships.
`Pass:` A message containing `{{first_name}}`, `null`, or `undefined` is reported as a defect without my writing that assertion.

**515. [P1]** As a tester, I want the agent to extract an OTP or magic link and continue the flow, so that authentication journeys complete.
`Pass:` The agent pulls the code or link from the message and uses it in the browser to finish the journey.

**516. [P1]** As a tester, I want links in the message verified, so that dead or wrong links are caught.
`Pass:` Links are followed and reported by status; a link to the wrong environment (e.g. staging URL in a prod message) is flagged.

**517. [P1]** As a tester, I want single-use and expiring links tested, so that security behaviour is proven.
`Pass:` The agent verifies a reset link works once and is rejected on reuse or after expiry.

**518. [P2]** As a tester, I want attachments verified, so that document-bearing notifications are covered.
`Pass:` Attachment presence, filename, type, and basic content are asserted.

**519. [P2]** As a tester, I want message content checked across locales, so that translated notifications are correct.
`Pass:` The agent verifies the message arrives in the expected language for a locale-specific user.

---

## BH. The negative and timing cases — where the bugs are

**520. [P0]** As a tester, I want to assert that a message was *not* sent, so that I can prove opt-outs and suppression work.
`Pass:` "Unsubscribe, trigger the event, confirm nothing arrives" passes only after a stated quiet period with no message.

**521. [P0]** As a tester, I want duplicate sends detected, so that idempotency failures surface.
`Pass:` If two identical messages arrive for one trigger, the run reports a duplicate rather than passing on the first.

**522. [P1]** As a tester, I want delivery time measured against an SLA, so that "arrived eventually" isn't mistaken for "arrived on time".
`Pass:` Delivery latency is reported and can be asserted against a threshold.

**523. [P1]** As a tester, I want ordering verified when several messages are expected, so that sequence bugs are caught.
`Pass:` Multiple expected messages are asserted in order; out-of-order arrival is reported.

**524. [P1]** As a tester, I want the agent to distinguish "not sent" from "sent but not delivered", so that I know where the failure is.
`Pass:` Where provider signals allow, the report separates never-dispatched from dispatched-but-undelivered, and says when it can't tell.

**525. [P1]** As a tester, I want spam and deliverability signals reported for email, so that a "working" notification that nobody receives is caught.
`Pass:` The agent reports spam-folder placement where detectable, and SPF/DKIM/DMARC status on the received message.

**526. [P2]** As a tester, I want messages triggered by a background job or schedule to be waitable, so that delayed notifications are testable.
`Pass:` A long wait window can be configured, and the run can be resumed or checked later without holding a browser open.

**527. [P2]** As a tester, I want channel fallback behaviour tested, so that multi-channel logic is proven.
`Pass:` "If SMS fails, email should be sent" is assertable across both inboxes in one run.

---

## BI. Connections — the management UI

**528. [P0]** As a tester, I want one place to see all my connections and their status, so that I know what the agent can reach.
`Pass:` A connections page lists each integration with a clear connected / disconnected / error state and when it was last verified.

**529. [P0]** As a tester, I want a "test connection" button, so that I can confirm a connection works before relying on it in a run.
`Pass:` The test performs a real round-trip (sends and reads a message where applicable) and reports success or the specific failure.

**530. [P0]** As a tester, I want connection setup to be a few plain steps, so that I don't need documentation.
`Pass:` A new email, SMS, or WhatsApp connection is completed in under five steps with plain-language labels, no protocol jargon required.

**531. [P1]** As a tester, I want failures explained in terms I can act on, so that I can fix them myself.
`Pass:` Errors distinguish bad credentials, expired token, revoked access, wrong permissions/scope, rate limit, quota exhausted, and provider outage — each with the next step.

**532. [P1]** As a tester, I want connections scoped to a project, so that one team's integration isn't silently used by another.
`Pass:` A connection belongs to a project (or is explicitly shared org-wide); scope is visible on the connection.

**533. [P1]** As a tester, I want to disable a connection without deleting it, so that I can pause an integration temporarily.
`Pass:` Disabling makes it unavailable to the agent while preserving its configuration; re-enabling needs no reconfiguration.

**534. [P1]** As a tester, I want to know what breaks before I delete a connection, so that I don't silently break saved tests.
`Pass:` Deletion lists dependent tests and suites and requires confirmation; affected tests are flagged rather than failing mysteriously later.

**535. [P1]** As a tester, I want connection health checked proactively, so that I find out before a run fails.
`Pass:` Expiring or broken connections are surfaced on the connections page and warned about before a run that needs them.

**536. [P2]** As a tester, I want to add other MCP servers beyond the built-in channels, so that the system extends to our own tools.
`Pass:` A generic MCP server can be added by endpoint and credentials; its tools appear to the agent and are listed on the page.

**537. [P2]** As a manager, I want to see which connections are used by which tests, so that integrations are governable.
`Pass:` Each connection shows its usage across tests and recent runs.

---

## BJ. Setting up through the agent

**538. [P1]** As a tester, I want the agent to offer to set up a connection when a task needs one, so that I'm never stuck at a dead end.
`Pass:` Asking to test notifications without an inbox configured opens the setup flow inline, then resumes the original request once connected.

**539. [P1]** As a tester, I want the agent to tell me exactly which connection is missing, so that I know what to fix.
`Pass:` "This needs an SMS inbox for this project" — not a generic "I can't do that."

**540. [P1]** As a tester, I want a held request to resume itself after I connect, so that setup feels like continuing rather than restarting.
`Pass:` After a successful connection, the original test runs with its parameters intact.

**541. [P1]** As a tester, I want to ask the agent what it can currently reach, so that I understand its capabilities without hunting through settings.
`Pass:` "What integrations do I have?" returns the connected channels and their status in plain language.

**542. [P2]** As a tester, I want the agent to suggest the simplest channel when I'm vague, so that I'm not forced to choose blind.
`Pass:` "Test the notification" prompts a short choice of available channels rather than guessing.

---

## BK. Safety, cost and credentials

**543. [P0]** As a security reviewer, I want integration credentials stored in the vault, never as plain config, so that connections aren't a breach vector.
`Pass:` API keys, tokens, and mailbox passwords are vault-held; they never appear in logs, evidence, exports, or the UI after entry.

**544. [P0]** As a security reviewer, I want the agent blocked from messaging real people, so that testing can never spam customers.
`Pass:` Sending to any destination other than a configured test identity is refused; an allowlist governs permitted destinations.

**545. [P1]** As a tester, I want SMS and WhatsApp spend visible and capped, so that a looping test doesn't run up a bill.
`Pass:` Per-message cost and running total are shown; a project cap halts sending at the limit.

**546. [P1]** As a tester, I want received message content redacted in shared evidence, so that OTPs and personal data don't leak into reports.
`Pass:` Codes, tokens, and PII in captured messages are masked in reports and exports by default.

**547. [P1]** As a tester, I want a sandbox mode where nothing leaves the system, so that I can build tests safely before going live.
`Pass:` Sandbox mode is clearly indicated and no real messages are dispatched while it's on.

---

## BL. Video evidence

> Keep it simple: one recording per run, synced to the step list, jumpable.

**548. [P0]** As a tester, I want a video recording of each run, so that I can see what happened rather than reconstruct it from screenshots.
`Pass:` A run produces a playable recording of the browser session, available in the run's evidence.

**549. [P0]** As a tester, I want the video synced to the step list, so that I can jump straight to the moment that matters.
`Pass:` Clicking a step seeks the video to that timestamp; playing the video highlights the current step.

**550. [P0]** As a tester, I want failures auto-marked on the timeline, so that I find the problem without scrubbing.
`Pass:` Each failure appears as a marker on the video timeline; clicking it jumps there.

**551. [P1]** As a tester, I want idle time skipped or compressed, so that a long run isn't a long video.
`Pass:` Waiting periods are compressed or skippable; playback speed is adjustable; the true elapsed time is still shown.

**552. [P1]** As a tester, I want to export a short clip for a single defect, so that a bug report carries a video a developer will actually watch.
`Pass:` A clip covering one failure exports and attaches to the defect, with the surrounding steps included.

**553. [P1]** As a tester, I want console and network errors visible alongside the video, so that cause and effect line up.
`Pass:` Errors are shown on the same timeline as the video and are correlated to the step that produced them.

**554. [P1]** As a tester, I want credentials and PII masked in the recording, so that a shared video isn't a leak.
`Pass:` Password fields and flagged PII are obscured in the video itself, not only in the report.

**555. [P1]** As a tester, I want a failed recording never to fail the run, so that evidence capture can't cost me a result.
`Pass:` If recording fails, the run completes normally and the report notes that video is unavailable.

**556. [P1]** As a QA lead, I want recording configurable and retention bounded, so that storage stays manageable.
`Pass:` Recording can be set to always, on-failure-only, or off, per project; retention is configurable and expiry is stated on the evidence.

---

## Build order

**Start with email (504, 505, 511, 512).** One test address with per-run uniqueness, plus arrival assertion with a timeout, covers the majority of real notification testing. SMS and WhatsApp are the same shape once the inbox abstraction exists — build the abstraction first so adding a channel is configuration, not a rewrite.

**Then the negative assertion (520).** It's the story that separates you from "we can check an email arrived." Proving that *nothing* was sent when a user opted out is what a real QA team needs and almost nothing does well.

**Then test-connection (529) and honest errors (531).** Integrations fail constantly for boring reasons — expired tokens, wrong scopes, quota. A test button plus an error message that names the actual cause is the difference between a feature people trust and one they abandon.

**For video, 548–550 are the whole feature.** Recording, step-sync, and failure markers deliver nearly all the value. Clip export and console correlation are the polish. Do not let recording failures fail runs (555) — that's the one edge case that turns a nice-to-have into a liability.
