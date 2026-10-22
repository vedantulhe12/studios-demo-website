# Benchmarks

Generated 2026-09-11T21:27:46.804Z from `evals/benchmark/results/benchmarks.jsonl`. Detection is against seeded ground truth; **false positives sit next to it on purpose**. Rows under 10 finished cases are marked thin and are not compared.

| Pack | Model | Cases | Completion | Detection | False positives | First result | Cost / case | HITL | Runs (pass / fail / blocked / inconclusive) |
|---|---|---|---|---|---|---|---|---|---|

## What a pass means per pack

- **Form app (Acme policyholder / register)** — The agent explores the forms, finds the loose e-mail validation, and files nothing that is not a defect. _Limitations: no-captcha, vision-quality._
- **E-commerce (Acme quotes, coupons, reservations)** — Coupon stacking, the client-only minimum and at least one race are found with repro steps. _Limitations: race-timing, vision-quality._
- **Data dashboard (Acme claims table, EMI)** — Numeric and sorting defects are found from the rendered table, not guessed. _Limitations: vision-quality._
- **Auth and blocked targets** — Login walls, production targets, unreachable hosts and a Cloudflare challenge end as BLOCKED / ENV_ERROR with remedies — never as a pass and never bypassed. _Limitations: no-captcha, no-evasion, egress-changes._
- **Exploratory defect hunt (Acme, all modules)** — A bounded exploration finds the majority of seeded defects with a low false-positive rate and grows the app map. _Limitations: race-timing, vision-quality, exploration-cap._
- **Performance (web vitals, baselines)** — Vitals are measured, compared with a baseline and reported with thresholds, not adjectives. _Limitations: browserless-latency._
- **Accessibility (web + canvas + device)** — Unlabelled controls are found on every surface; contrast is stated as unknown where it cannot be computed. _Limitations: no-screen-reader._
- **Locale matrix (Acme, 3 locales)** — The same flow runs per locale as matrix cells and the report leads with the cells that disagree. _Limitations: engine-availability._
- **Volume (suite of 20+ cases as a campaign)** — A costed campaign is proposed, priced from history, split into tranches, and runs to completion through the queue. _Limitations: pricing-history._
- **Approvals and envelopes** — Envelope caps gate, chains decide, budget checkpoints pause and resume — every step audited.
- **Mobile fixture (DemoBank on Appetize / mock lane)** — Install, boot, sign in, find the seeded app defects, and say INCONCLUSIVE where the lane cannot observe. _Limitations: appetize-no-multitouch, ios-no-network-conditions, one-build-per-session._
- **Canvas Lite (desktop worker)** — Through a paired worker the agent finds the seeded creative-app defects with the method (accessibility / pixel / script oracle) named. _Limitations: worker-required, macos-tcc, windows-unvalidated._
- **Cross-surface journey (web + Android)** — One journey runs as isolated arms and the parity report names divergent steps and strings. _Limitations: appetize-no-multitouch, engine-availability._

Run: `npm run benchmark -- --pack <id>|all --models google/gemini-3.7-flash,z-ai/glm-5.3` against a running API with an OpenRouter key. Results append to the JSONL store; this file is regenerated each run.
