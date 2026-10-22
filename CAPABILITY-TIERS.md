# Capability tiers

What runs today, on which lane, and what each lane cannot do. Generated from the code (`npm run benchmark -- --render`).

## Web

| Capability | Tier | Notes |
|---|---|---|
| Live Chromium via Browserless | shipped | Every chat and run; frames streamed to web and phone |
| Firefox / WebKit / Chrome / Edge channels, profiles, extensions, headed | shipped (local engine) | Worker or API host with `LOCAL_BROWSERS=1`; a missing engine fails its cell truthfully |
| Device emulation, geolocation, permissions, colour scheme, offline | shipped | `emulate`; recorded on `run.build` |
| Network / storage / clock / flag manipulation with audit | shipped | Changes tab on every run |
| Matrix runs (engine × viewport / device) | shipped | Cells are durable jobs; divergent cells lead the report |
| Blocked-target classification and remedies | shipped | Never evades; BLOCKED verdict with lanes |
| Pinned older browser versions | connector tier | Not bundled |

## Mobile

| Lane | Tier | Multi-touch | Install over existing | adb | Network conditions | Push | Biometrics | Network capture | Real hardware |
|---|---|---|---|---|---|---|---|---|---|
| appetize | virtual | no | no | yes | android | android | yes | yes | no |
| local | local | yes | yes | yes | android | all | yes | no | no |
| mock | fixture | yes | yes | yes | android | android | yes | yes | no |
| real-device farm (BrowserStack / Sauce / USB) | connector tier | interface shipped (`DeviceProvider`); lane not bundled | | | | | | | |

- **appetize**: Tap and single-finger swipe only (no pinch); One build per session; upgrade-over-existing needs a real-device or local lane; iOS: no network conditions, notifications, battery or font scale; lock toggle only; adb output is not returned on the public cloud — effects are verified from the screen
- **local**: Real multi-touch: Android through the emulator gRPC controller, iOS through WebDriverAgent (XCTest) — pinch and two-finger rotate; Clean device per session: Android boots the AVD read-only; iOS uses a KaDeep-owned simulator with the app reinstalled; Install over an existing build keeps data (adb install -r / simctl install); No intercepted network traffic (Appetize has a proxy; local does not); iOS: macOS with Xcode only; no network conditions or battery; push via simctl; Android: not on Windows on ARM; images must match the computer (arm64 on Apple Silicon, x86_64 on Intel/AMD)
- **mock**: Deterministic fixture app; no real device

## Desktop

| Capability | Tier | Notes |
|---|---|---|
| Paired local worker (macOS / Linux; Windows builds, unvalidated) over cua-driver | shipped | Accessibility-first with pixel fallback; method logged per action |
| Overlay HUD, global stop, take control, headless worker, auto-update | shipped | Electron app |
| Cloud desktop VMs (cua Fleets / Lume) | roadmap | Flagged "coming soon" in the picker |
| Signed installers | entitlement | Needs Developer ID / code-signing certificates in the workflow secrets |

## Connectors (MCP presets)

| Preset | Transport | Status | Use for |
|---|---|---|---|
| GitHub | http | live | File defects as issues, read the PR a release candidate came from, check CI status before a campaign. |
| Jira (Atlassian) | sse | live | Import requirements from epics, file and update defects, link runs to tickets. |
| Slack | stdio | live | Post campaign summaries and approval links; read acknowledgements in the QA channel. |
| Railway | stdio | live | Check the build a run tested, read deploy logs when an ENV_ERROR appears, confirm a preview environment is up before a campaign. |
| MongoDB | stdio | live | Verify what a UI action really wrote (order status, audit rows) — the same job as db_verify, with schema exploration. |
| Adobe Firefly / Creativity | http | entitlement (mock-tested) | Seed test assets, verify generated output metadata, exercise Creative Cloud round-trips from a QA flow. |
| Adobe Analytics | http | entitlement (mock-tested) | Assert that a tested journey produced the expected analytics hits; compare funnels before and after a release. |
| Adobe Customer Journey Analytics | http | entitlement (mock-tested) | Cross-check cross-surface journeys against the analytics view of the same journey. |
| Adobe Experience Manager | http | entitlement (mock-tested) | Confirm the content a page should render, publish a test fragment, verify authoring → publish parity. |

## Governance

- Spending envelopes, costed campaigns, tranches, approval chains, budget checkpoints, break-glass, standing approvals — shipped; Slack / Teams approvals via signed action links (interactive buttons need a Slack app).
