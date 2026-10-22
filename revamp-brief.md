# Master prompt: project setup and project charter revamp (QA first)

Refined from the dictated brief in [`revamp-brief-raw.md`](./revamp-brief-raw.md) (blocks R1–R9) plus the owner's decisions of 2026-10-18 (§10). Grounded in the repo at commit `d289a96`, branch `feat/product-charter` (on latest `dev`). Line numbers were read at that commit; re-check them before editing.

**For the session that picks this up.** Read this file, `CLAUDE.md`, `docs/ARCHITECTURE.md`, then the files named in §5. All questions are answered; §10 is the decision log and §11 lists the places where a decision meets a technical constraint you must handle in the plan. Start with Phase 0 (§9): write the plan, get it approved, then build phase by phase on `feat/product-charter` with the checks after each phase. Work like a senior engineer who knows agentic QA, evals and harnesses; UI decisions are made by a senior product designer; agent decisions by a senior agentic engineer. The agent is the most valuable thing in this repo: touch it last, touch it least, and measure before and after.

---

## 1. Context

KaDeep Studios is moving to selling directly to clients. In sales meetings the demo jumped straight to execution ("type a prompt, the agent runs a test case") and clients were not convinced. The agreed strategy with the sales team: a client first sets up the project in full, receives a project charter, and only then reaches the main workspace, where a short walkthrough of the product follows.

The setup flow exists today as a five-step overlay (Project, Scope, Connections, Rules, Review) in `apps/web/src/ProjectCharter.tsx`. It is a UI that writes a few settings; the agent does not know most of what the client typed, nothing is uploaded, nothing runs after creation, and the review page is a static card. This brief is a full-stack revamp of that flow: the UI, the backend, and the way the result reaches the agent.

## 2. Goals

1. A client who finishes setup believes the product works, end to end, in a proper flow. The bar is the first-run setup of US SaaS products (Adobe, Cursor, Arc, Dia): smooth, complete, nothing half-wired.
2. When the client lands in the workspace, the agent already knows this client's product: what it is, how it flows, what was uploaded (BRDs, legacy test cases, builds), which tools are connected, which rules apply, and what the team wrote about where the product is solid and where it is fragile. The agent takes this from the project setup only, per project, per client.
3. The agent's current accuracy (the owner puts it at 90–95%) is not reduced. Setup knowledge reaches the agent through its existing seams (a prompt block, a tool, a project skill) and is measured with evals before and after.
4. The revamp is UI/UX and functionality together: components, modals, icons, SVGs, tables, spacing, and the backend that makes each control real.

## 3. Scope

### The flow after this work (six steps)

1. **Project & Scope** — type, name, URL(s), platform matrix, environments, versions, coverage matrix.
2. **Uploads** — optional, its own page.
3. **Connections** — the client's tools, databases, CI/CD.
4. **LLM** — the model/provider the agent uses, and the CLI lane.
5. **Rules** — four real toggles plus "what the agent should know about your platform".
6. **Review** — the animated project charter → Create project → setup job + loader → workspace → first-time walkthrough.

Nothing in steps 2–4 is mandatory; everything there is also available in Settings afterwards.

### In scope

- The QA / testing setup, all six steps, draft lifecycle, what happens immediately after "Create project", the first-time walkthrough.
- Backend: structured charter on the project, rule toggles wired to behaviour, charter prompt block + tool + always-on project skill, the setup job, real database connections with enforced read-only, the MCP credential fixes, multi-provider LLM connections (handled as the riskiest change, §7.6).
- Loc-safe refactor of the shared wizard shell (§4, decision 13).

### Deferred (next brief), but design for it

- Localization setup: loc-specific uploads (SCORM, media, XLIFF, PDF, XML, EPUB, websites/single pages, PNG/JPEG), loc rules, the loc branch of each step. Do not ship loc-only uploads or new loc rules here. Do not create a QA-only data model that loc cannot reuse.
- Cross-browser / cross-configuration campaigns that run the whole matrix automatically (decision 19).
- Walkthrough for admin/manager personas on `EnterpriseShell` (decision 17).
- OAuth apps registered with the big providers (decision 3).
- SQL Server / MySQL / Oracle connections (shown as Coming soon, decision 6).
- Screenshot redaction (decision 8).

### Out of scope

- Changes to the drivers or the execution plane beyond what the setup job needs.
- `apps/mobile` project creation (it has its own form, `apps/mobile/app/projects.tsx`).
- `apps/api/src/loc`, `apps/api/src/video`, `services/*` (video worker, memory sidecar) and loc product code: not modified.

## 4. What must not change or break

- **Agent accuracy.** Baseline the eval stories in §5.8 on the unchanged branch before the first agent-touching commit; re-run after every agent-touching phase; a drop blocks the phase. No change to `runLoop()` semantics, tool contracts or prompt order that is not covered by that comparison.
- **QA mode and loc mode stay separate.** `apps/web/e2e/zz-qa-mode-unchanged.spec.ts` and `npm run test:e2e:loc` stay green. Loc behaviour is unchanged this phase (same kind switch, locales, content types, skip-connect, create). The shared shell in `ProjectCharter.tsx` may be refactored and the file may be broken apart; the loc branch is not frozen byte-for-byte. When steps or testids move, update `apps/web/e2e/loc-helpers.ts` (`openCharter`, `finishCharter`, `makeLocProject`, `makeQaProject`) in the same commit and say so in the PR. Today `finishCharter` is: next → wait for inbox tile → next → next → create.
- **Architecture seams in `CLAUDE.md`:** drivers own the runtime; API tools stay thin; `services/llm-keys.ts` is the only place a model key is resolved (extend it, do not bypass it); skills are routed by frontmatter and `selectSkills` only; governance seam is one-way; `collection()` is the store accessor; credentials live in the vault and never in memory or the prompt (the prompt may name `vault:…`, never the value).
- **Existing settings surfaces** (Settings › Connect, Providers, Browsers, Devices, Rules) keep working; the wizard writes through the same services and routes.
- **Typecheck is the lint gate.** `npm run typecheck` clean after every phase.
- **Honesty conventions:** options that do not work are "Coming soon" or disabled with a reason; loader lines describe work that already finished; a failed connection test says failed; a floor that cannot be switched off is stated under the control, not hidden behind a switch that lies.
- **Brand:** "KaDeep Studio Pro" lockup (`BrandMark.tsx`) unchanged. Dark is the default theme; both themes must look finished.

## 5. Current state (what the code says)

Read these before planning. **Gap** marks where the dictated brief assumed something the code does not do.

### 5.1 Where the flow lives

- `apps/web/src/ProjectCharter.tsx` (515 lines): steps, state, `rulesText`, review card. Styles `apps/web/src/projectCharter.css` (922 lines). Sub-components `apps/web/src/charter/PlatformMatrix.tsx`, `apps/web/src/charter/ConnectionsStep.tsx`.
- Not a route: a full-screen overlay driven by `creating` state in `apps/web/src/App.tsx:250`, mounted at `App.tsx:1523-1535`. Entry points: `projects-home-new`, sidebar `new-project`, `loc-rail-new-project` (opens on Testing, not Localization — `App.tsx:2130`), MidPane `entry-test` / `entry-localize`, sending a chat with no project (`sendAfter`).
- A "draft" is a real project row created by `POST /api/projects` when leaving step 1 (`App.tsx:961`). Cancel/Escape deletes it. **No draft flag on the server**; a closed tab leaves a normal project in the list.
- `apps/web/src/ProjectDialog.tsx` is edit-only; the charter imports `parseAppUrl` and `ProjectFields` from it.
- Desktop (`apps/desktop`) renders the same web UI.

### 5.2 What "Create project" does today (`charterFinish`, `App.tsx:974-992`)

1. `PATCH /api/projects/:id` with one `environment` (staging if ticked, else the first ticked).
2. `PATCH /api/projects/:id/settings` with `conventions` (the whole charter as markdown, `ProjectCharter.tsx:157-177`), `runSettings.browser` / `viewport` (first browser and first size only), `runSettings.readOnly` (only when the production toggle is on **and** environment is exactly `production`), `browsers.target`.
3. `PATCH /api/projects/:id/devices` with `mobile.runOn`.
4. `POST /a11y/schedule` if "Accessibility" was ticked.
5. Select the project, toast, send the pending chat message if any.

**Gap:** test types, journeys, device list, extra browsers/sizes, env list, desktop OS, connected tools and the notes box exist only as text in `conventions`. `Project` (`packages/shared/src/index.ts:1265-1327`) has no `charter`, `scope`, `rules` or `notes` field. Nothing runs on the server after creation; no setup/onboarding job kind exists (`shared/index.ts:110`; `jobs/worker.ts:142-167`, where `exploration` is declared but not runnable).

### 5.3 How the agent sees the project today

- `buildSystem()` in `apps/api/src/agent/runtime.ts:411-464` assembles: voice, mode, project, artifacts, decisions, continuity, product model, memory, skills, rules.
- `projectBlock()` (`apps/api/src/agent/prompts.ts:55-72`) reads name, baseUrl, description, environment (PRODUCTION warning on `/prod/i`), variables, note titles, readOnly, allowedHosts, browser, budget, `mobileLine()`, and `conventions` in full.
- Runs use `runnerSystem()` (`prompts.ts:117-133`), which **cuts conventions to 1,200 characters** (`:132`). No cap in the settings route; `set_conventions` tool caps at 4,000.
- No per-block budget; compaction never trims the system prompt (`packages/agent/src/loop.ts:159-163`). Context window default 128k (`runtime.ts:210,253`).
- The `Notes:` line in the prompt lists the notes collection, not the wizard's notes box.

### 5.4 Rule toggles vs. behaviour today

| Toggle (`ProjectCharter.tsx:62-67`) | Today |
|---|---|
| Never change data on production | Sets `runSettings.readOnly` only when env = `production`; the default staging plan never sets it. When set, writes are blocked at three levels: browser-driver aborts non-GET (`packages/browser-driver/src/index.ts:734-741`, via `blockWrites` in `agent/context.ts:454`), browser tool guard (`agent/tools/browser.ts:203-215`), mobile guard (`tools/mobile.ts:162-165`). The rules text says "(production is read-only)" **unconditionally** (`ProjectCharter.tsx:165`). |
| Ask before anything destructive | Text only. The destructive guard (`browser.ts:218-219`, regex `:33`, bypass `confirm_destructive`; `mobile.ts:166-167`; `desktop.ts:377`) is always on and ignores the toggle. `services/approvals.ts` is the org approval chain for campaigns/budgets — unrelated. |
| Mask personal data in reports | Text only. `maskPii()` is always on for snapshot text (`browser.ts:61-88`): cards, Aadhaar, PAN; emails/phones deliberately left; screenshots untouched. The hint "Names, emails and card numbers are hidden" overstates this. |
| A person signs off every release | Text only. Sign-offs exist (`services/releases.ts:57-87`; readiness gate `services/dashboard.ts:137-140`) but nothing reads the toggle. |

### 5.5 Skills, product model, memory, uploads, builds

- **Skills.** `selectSkills()` (`apps/api/src/services/skills.ts:213-272`): lead always (`qa-lead` / `loc-lead`, plus `mobile-qa-lead` / `desktop-qa-lead` by surface), keyword scoring against `when`/description, top 3–4, then `routes` / `precedes` / `follows`. Project skills: Mongo `skills` collection by `projectId` (written by `create_skill`, `POST /api/projects/:id/skills`, pack import `services/skill-packs.ts`), +1 in scoring, rendered `## Skill: <name>`. There is no "always include" flag for project skills. `skills/product-model-bootstrap/SKILL.md` is the existing day-one recipe (`import_requirements` → vault logins → bounded `start_exploration` with writes blocked → `remember(app)` → notes + first smoke suite); it runs only when the message matches its `when` keywords. `skills/explore-and-learn` is the breadth-first site walk.
- **Product model** (`services/product-model.ts`): per project per env, filled only by `observeScreen()` after browser tool calls; flows/quirks writers test-only; rendered by `productModelBlock`. **Gap:** nothing seeds it; the `test-suite` role hint "seeds the product model" is not implemented.
- **Memory** (`apps/api/src/memory/`): layers `app:<projectId>:<env>`, `user:<userId>`, `team:<teamId>:<projectId>`; `writeFact()` (`memory/store.ts:111-185`) behind `guardMemory()` (`memory/guard.ts`: refuses secrets, scrubs cards/emails/phones/SSN); `recall()` is message-dependent, 14 facts. Per decision 9 the wizard will **not** write to memory.
- **Uploads.** `parseAttachment()` (`services/attachments/index.ts:195-300`): images, mobile builds, xlsx/xls/ods (all sheets), docx, pdf, json (incl. Postman), csv/tsv, text. `saveUploadedArtifact()` (`services/artifacts.ts:322-401`): 25 MB cap, secret-looking files refused, role guessed, `.extract.txt` written; routes `POST /api/projects/:id/artifacts/upload[-file]`. Requirements via `importRequirements()` / `POST /requirements`. Agent sees ≤25 files + ≤4 pinned texts via `artifactsBlock()` (`agent/artifact-context.ts:148-186`); tools `read_attachment`, `read_artifact`, `list_artifacts`, `import_requirements`, `list_requirements`, `create_flows`, `save_dataset`. Web picker accept list in `apps/web/src/files.ts:24`. **Gap:** no upload step in the wizard.
- **Builds.** `saveMobileBuildArtifact()` (400 MB) + `POST /api/projects/:id/mobile/builds` → `installBuild()` (`services/mobile-builds.ts:205-271`) → `project.mobile.targets[]` (`MobileTarget`: platform, buildId, appId, versionName, device, osVersion, sha256…). `.ipa` / `.aab` refused (`mobile-build-inspect.ts:123-128`). Appetize upload, or local-only for paired-machine platforms / no Appetize token. **Gap:** not reachable from the wizard.

### 5.6 Connections, MCP, LLM, CLI, pairing

- **No OAuth flow exists anywhere.** Every tile in `ConnectionsStep.tsx:50-116` is a pasted credential, a pasted webhook URL, a one-click built-in, or a generated token. Mechanisms today:
  - Jira / GitHub Issues / Linear → project variables + vault, tested by `POST /trackers/test`; used by `create_ticket` (`services/tickets.ts`). **Bug:** readiness reads `vars.GITHUB_TOKEN` but the wizard stores it as `VAULT_GITHUB_TOKEN` (`services/readiness.ts:65-66` vs `vault.ts:219-220`).
  - Slack / Teams → incoming-webhook rows in `connections` (outbound only).
  - Test inbox → built-in SMTP receiver; **needs `INBOX_SMTP_PORT` / `INBOX_DOMAIN`** on the API host or "Create a test inbox" fails (`services/connections.ts:530`); neither is in `.env.example` or Railway config.
  - SMS (Twilio) → pasted SID/token/number; webhook URL copied into Twilio by hand.
  - GitHub / MongoDB / Railway "Agent tools" → per-project `mcp_servers` rows (`routes/authoring.ts:343-361`). **Bugs:** stdio env (`MDB_MCP_CONNECTION_STRING`, `RAILWAY_API_TOKEN`) is used for the probe and **not stored** (`McpServerConfig` has no `env`; `mcp/client.ts:71,151-153`); servers are keyed by name globally (`client.ts:71`), so two projects named `github` collide.
  - CI/CD → `POST /ci-token` mints `ts_…` stored plaintext on the project; authorises `/api/ci/:projectId/run`, a11y and loc CLI routes.
- **Databases.** No MongoDB / Postgres / SQL connection kind (`ConnectionKind` = email | sms | whatsapp | slack | teams | mcp). `db_verify` (`agent/tools/network.ts:320-362`) looks for providers that cannot be created. `pg` is an optional, unlisted dependency; the API depends on `mongodb`.
- **MCP.** API is an MCP client (operator `mcp.json` + per-project rows; stdio / streamable HTTP / SSE; tools `mcp_<server>_<tool>`) and an MCP server (`POST /mcp`, `POST /mcp/agent` for desktop CLIs). `GET /api/mcp-presets` is not called by the web app. Settings › Connect "MCP server" form has no env field.
- **LLM.** Agent loop (`packages/agent`, `runLoop()`) speaks OpenRouter's OpenAI-compatible chat-completions API only. Key is **per user**, pasted, verified (`services/llm-keys.ts`), with operator `OPENROUTER_API_KEY` as fallback. Direct Anthropic key exists only for Video Translate (`@anthropic-ai/sdk`). Model per turn: `project.models.lead` → org default → env default, with org allowlist (`runtime.ts:199-201,378-397`). No provider sign-in exists. Model catalog is OpenRouter's.
- **CLI lane.** Codex / Claude Code / Cursor / OpenCode run on a **paired desktop** with the user's own terminal login; surfaced in Settings › Providers, picked in the composer (`agent/cli-turn.ts`, `routes/providers.ts`, `docs/releases/agent-clis.md`). No OpenRouter key needed for those turns.
- **Desktop pairing** by code or deep link (`kadeepstudiospro://worker/pair…`), no QR. "Your desktop app", "Emulator on your computer", "Simulator on your Mac" and the Desktop platform need a paired machine; local devices also need org flag `features.localDevices` (`routes/mobile.ts:188-199`). Desktop OS choice is not saved.

### 5.7 UI infrastructure

- Motion: `motion` v13 (`motion/react`) in the charter (`AnimatePresence` slide + blur; fade under reduced motion); CSS keyframes elsewhere. No three.js / WebGL code in `apps/web` today.
- Reduced motion is a convention (`prefers-reduced-motion` blocks; `useReducedMotion()`); `KSMorphLoader` / `BrandLoader` ignore it, `video/KsMark.tsx` wraps it correctly.
- Loaders: `KSMorphLoader.tsx` (canvas dot-morph KS → spinner → … → check), used by `BrandLoader` (auth) and `video/KsMark`. Not used by the charter.
- Tokens: `--bg`, `--accent`, `--radius`, `--font`… in `apps/web/src/styles.css`; body 12 px after `d289a96`; system font stack (Inter loaded but unused).
- Icons: `icons.tsx` (Chrome/Firefox/Safari hand-drawn), `providerLogos.ts`. **Charter tiles show two-letter initials** for every brand except Microsoft (`ProviderGlyph`). Real marks exist in `ui/BrandLogo.tsx` (simple-icons), used in Settings only.
- After creation `selectProject()` opens the last page for the kind (`browser` for QA). **Admins/managers get `EnterpriseShell`, not the workspace** (`App.tsx:1606,1993-1996`).
- No tour / walkthrough / welcome state exists. `Spotlight.tsx` is ⌘K search.

### 5.8 Evals and tests that touch this

- Evals: `088-project-context` (conventions reach a new chat), `104`, `272` (read-only), `136` (standing rules), `008`, `103` (prod env), `102` (destructive confirm), `106` (PII flag), `156` (cold start). Run: `npm run dev:api`, then `npm run evals -- --stories 8,88,102,103,104,106,136,156,272`. Last recorded run covered only 278–284; `docs/BENCHMARKS.md` has no rows.
- API tests: `loc-project.test.ts`, `a11y-schedule.test.ts:117`, `local-devices.test.ts`, `a11y-ci.test.ts`, `ci-async.test.ts`, `mobile.test.ts:599-610` (prompt wording). **None covers the generated `rulesText` or the wizard end to end.**
- E2E testids in use: `project-charter` (+`data-kind`, `.is-loc`), `charter-kind-qa|loc`, `charter-name`, `charter-url`, `charter-next`, `charter-create`, `charter-close`, `charter-platform-web`, `charter-source`, `charter-target-locales`, `charter-target-chip-<tag>`, `charter-target-locale-error`, `charter-content-elearning`, `charter-connect-inbox`; entry points `projects-home-new`, `new-project`, `project-switch`, `entry-test`, `entry-localize`.

## 6. UI/UX requirements

### 6.0 Across every step

- US SaaS product feel: UI, UX, modals, components, icons, SVGs, tables, spacing. References: Adobe, Cursor, Arc, Dia first-run setups. Copy in en-US. Placeholders global (e.g. "Acme banking"), not KYC/India-only.
- Smooth, no flakiness: every control is real or marked unavailable with a reason; every async action has a visible state; errors say what to do next.
- Real brand marks for every integration and browser (`ui/BrandLogo.tsx` / simple-icons), not initials or hand-drawn approximations.
- Both themes; reduced motion honoured everywhere including loaders and the 3D charter; keyboard flow kept (Enter continues, Escape cancels).
- Layouts that work today may be kept or replaced; the standard above decides, not continuity.

### 6.1 Step 1 — Project & Scope (one page)

- Project type (Testing; Localization visible, its branch unchanged this phase), name, app/site URL(s).
- **Platform matrix with every permutation:** Web (Chrome, Edge, Firefox, Safari; cloud browser vs. the user's desktop app; screen sizes), Android (cloud device vs. emulator on the user's computer; which devices, how many), iOS (cloud device vs. Simulator on a Mac; which devices), Desktop (which OS). Keep and complete what `PlatformMatrix.tsx` already has. Options needing a paired machine / org flag / local engine stay selectable where pairing can follow, disabled with the reason where it cannot; never a silent downgrade (same rule as `ensureDevice()`).
- **Environments:** dev / staging / production; staging on by default; coverage plan built against the chosen environments.
- **Versions — all of them, in the matrix:** the uploaded build's version (from the build file), the release version the cycle targets (entered), OS versions per device (from the device provider's real list), and the browser version (reported by the runtime when known). Nothing invented: a version the product cannot read is shown as unknown, not guessed.
- The **coverage matrix** is shown on the page and represents exactly what the client chose.
- **Persistence:** the full matrix and every choice are stored as structured data on the project (new shared types) and the agent reads them (§7). `conventions` keeps being generated for compatibility. The full configuration list is saved and shown; runs are **not** auto-multiplied across it (decision 19).

### 6.2 Step 2 — Uploads (separate page, optional)

- "Upload everything you have, or skip": BRDs and requirement documents, Excel/CSV test cases, legacy test suites (5–10 years of cases), PDFs, builds (APK, iOS `.app.zip`; `.ipa` / `.aab` explained as not runnable), images. Formats parsed this phase: xlsx/xls/ods, csv/tsv, docx, pdf, json, md/txt, images, APK, iOS `.app.zip` (today's parser set). Anything else is accepted as a file only if the product can store and show it truthfully; otherwise it is refused with a reason.
- Through the existing artifact and build services so files appear in the workspace and to the agent (`artifactsBlock`, tools).
- Says plainly that uploading can be done later from the workspace.
- Loc-only formats (SCORM, XLIFF, media, EPUB…) are not shipped here; the page design must not prevent adding them.

### 6.3 Step 3 — Connections

- Groups as today: issue tracker (Jira, GitHub Issues, Linear), team chat (Slack, Microsoft Teams), inbox & messages (test inbox, SMS), agent tools (GitHub, Railway), **databases (MongoDB, Postgres live; SQL Server, MySQL, Oracle as Coming soon)**, CI/CD. Show the breadth of integrations the product supports at SaaS presentation level.
- **Mechanism (decision 3):** no OAuth agreements with the big providers in this phase. Each tile uses the provider's normal credential flow with as few clicks as possible — target ≤ 3: click Connect → the product opens the provider's token page for you (deep link to the exact settings page) and shows exactly what to copy → paste → tested → connected. When the provider requires visiting its own page to authorize, that redirect is acceptable as long as the result is a verified, persisted connection.
- Nothing is mandatory; every tile is also in Settings › Connect.
- Truthful state per tile: connected (with what the test proved) / failed (with next step) / not connected. Prerequisites the operator must set (e.g. the test inbox's `INBOX_SMTP_PORT` / `INBOX_DOMAIN`) are surfaced, not hidden behind a failing button.
- **Databases (decision 6):** the client's product database connects directly to KaDeep — URI stored in the vault, tested on save, a real connection kind, not only MCP. Read-only is **enforced by the product**: (a) on save, prove the credential cannot write (see §11.4 for how to probe without writing client data); refuse a URI whose user can write; (b) `db_verify` is limited to find/count (Mongo) or a single SELECT (Postgres) and is fixed to use this saved connection. If `db_verify` is not fixed, the DB tiles are not offered.
- **Fixes required in this step's backend:** MCP stdio `env` persisted (encrypted) and used on reconnect; MCP server names namespaced per project; readiness finds the GitHub token the wizard stored; CI token handling unchanged in behaviour.

### 6.4 Step 4 — LLM

- The client chooses what powers the agent for this organization:
  - **Bring your own provider / key.** Companies have their own setups (the owner named GitHub Copilot and a separate Claude setup). Offer the providers the company may already pay for, each as one tile: paste the key (or the provider's equivalent credential) → verified → connected. The plan fixes the provider list; at minimum OpenRouter (today), Anthropic direct, and the OpenAI-compatible family (OpenAI, Azure OpenAI); others where the plan can verify them cheaply. Copilot: see §11.2.
  - **Scope is global:** one connection serves every project of the organization. The existing per-user key path keeps working; precedence (project model choice → org connection → user key → operator env) is fixed in the plan and implemented inside `services/llm-keys.ts`.
  - **CLI lane:** surface the existing paired-desktop CLIs (Codex, Claude Code, Cursor, OpenCode) from this page, with the pairing path. Nothing new in this phase.
- ≤ 2 clicks after the credential is in hand; truthful verified state; which model the agent will use is visible.
- Nothing mandatory: the operator key, when present, keeps the agent working.

### 6.5 Step 5 — Rules

Four toggles, **all default on, each changing real behaviour; copy matches code; no toggle that only writes markdown** (decision 8):

| Toggle | On | Off | Floor (never off; stated under the control) |
|---|---|---|---|
| Ask before anything destructive | Today's guard: before delete / wipe / close-account the agent stops and needs `confirm_destructive`. | Skip that extra ask on staging/test so a run can finish. | Real card numbers, real payments, payment buttons when the run is production. |
| Mask personal data in reports | Also mask emails and phone numbers in the text log. | Only the floor; emails/phones stay (they are test data). | Card PANs, Aadhaar, PAN, SSN in the text log. Hint: "card numbers and government IDs in the text log; screenshots are not redacted." Never claim names are hidden. |
| Never change data on production | Read-only **only when this run's environment is production** (browser drops non-GET, submit tools blocked). Staging/test in the same project may still write. | Writes allowed on staging/test; production still warns; the payment/card floor still applies. | Do not lock the whole project read-only because production appears in the coverage matrix. |
| A person signs off every release | Release cannot go out without a human sign-off (existing readiness gate). | That gate is not required. | — |

Below the toggles, the free-text field "what the agent should know about your platform" stays here (decision 1): where the product is solid, where it is fragile, what to test carefully, standing rules. It reaches the agent in full (§7.3).

### 6.6 Step 6 — Review: the project charter

- Replace the static card with a **dynamic, animated charter**: every connection shown as a live link ("your Slack is connected to KaDeep Studio Pro", Jira, GitHub, database…), selected devices and platforms in motion, web/mobile/desktop coverage, versions, rules, uploads, and the agent's knowledge state. "Animated means something is working there, not static." Premium feel; every SVG/PNG clean.
- 3D and WebGL are allowed (decision 16); choose what serves the page. Must degrade to a complete, legible, static charter under `prefers-reduced-motion` and when WebGL is unavailable; bundle size and first paint are measured in the plan.
- Everything on the charter is checkable by the client.

### 6.7 Drafts (decision 14)

- A real `draft` flag on the project. Drafts are hidden from the main project list until Create project succeeds.
- Closing the tab / refresh resumes the same setup (files, connections, notes kept). "New project" while a draft exists reopens that draft; never a second one.
- Cancel deletes the draft (confirm if anything was uploaded). Unfinished drafts older than 7 days are deleted.
- Only a finished project starts the setup job (§6.8). Loc e2e: `finishCharter` still ends with a finished project; if a helper meets a leftover draft it resumes or deletes it (same commit).

### 6.8 After "Create project": setup job and loader (decision 11)

- The app does not open abruptly. A full-screen setup state covers the screen; **each line shown is work that already finished. No fake timers.**
- **Must run, blocks the loader, needs no LLM and no browser, ≤ 60 s:** save charter · parse uploads · store Artifacts · import requirements when a file is a BRD/sheet · register builds.
- **Must not run in the spinner:** full site exploration, Excel→Flows conversion, SCORM/XLIFF.
- **If an LLM key exists:** optional one short summarise call (≤ 45 s); optional read-only URL ping in the background (≤ 60 s, ≤ 15 actions). Neither blocks first open.
- **No key:** still finishes; skips summarise and ping; does **not** show "agent is analyzing". Copy: files and charter are saved — add an OpenRouter key to read the site.
- **Hard cap 90 s → open the workspace.** Leftover work continues as the same durable job ("still indexing…"). Closing the tab resumes the job. The job must not fail for a missing LLM key.
- Runs on the existing job queue as a new kind, with progress events to the client; resumable after a process dies.

### 6.9 First-time walkthrough (decisions 12, 17)

- Tester workspace only, the first time **that user** opens a finished QA project. 6–7 steps. Dismissable; reopen from Help/Settings. Admins/managers on `EnterpriseShell`: no tour this phase.
- The planning session chooses the 6–7 steps as a senior UX person would: the steps that give a new tester the most of the platform in one pass, each justified in the plan in one line (what the user can do after seeing it).
- Same US SaaS aesthetic as setup.

## 7. Agent changes

Principle: integrate through existing seams; keep the loop's behaviour; measure.

### 7.1 Source of truth is the prompt (decision 9)

- The charter is persisted as structured project fields (scope / coverage, rules, notes, connections summary, uploads index, versions).
- A **charter prompt block** renders it into **every chat turn and every run** — added to `buildSystem()` and `runnerSystem()` — with **no 1,200-character cut** on notes or rules. `conventions` stays generated for backward compatibility but is no longer the carrier for new projects.
- **Do not copy the notes into memory.** Memory stays for facts the agent learns while testing, behind the PII guard. Vault secrets stay in the vault; the prompt may name `vault:…`, never a value.

### 7.2 Charter tool and always-on project skill (decisions 10, R3)

- A tool the agent calls to retrieve the full charter (structured data, notes, uploaded-file index with roles, connection states) when the prompt block's summary is not enough.
- A per-project "product knowledge" skill generated at setup and refreshed when the charter changes, stored in the project `skills` collection like any other project skill; `selectSkills` gains a small `always` flag honoured for project skills so it is included every turn. Routing stays frontmatter-driven; no code names the skill.
- Where setup data allows, seed the product model and the uploads' roles (the `test-suite` role hint promises this today).
- If something is missing, the agent explores the product itself (existing `product-model-bootstrap` / `explore-and-learn` / `start_exploration`).

### 7.3 Rule toggles wired (decision 8)

Each toggle is a persisted field and drives the existing mechanism exactly as the table in §6.5 says: `runSettings.readOnly` / `blockWrites` scoped to runs whose environment is production; the destructive guard and `confirm_destructive` with the payment/card floor; `maskPii()` with a floor set and an on/off extension to emails/phones; the readiness sign-off gate. Unit tests per toggle, on and off.

### 7.4 Team notes

Reach the agent in every chat and run through the charter block, untruncated, retrievable through the charter tool. The agent is told they may not apply to every task and to say when a rule stopped it (today's Rules copy).

### 7.5 Setup job (decision 11)

Implements §6.8: deterministic steps first, optional model/browser steps only when a key exists and never blocking first open, durable and resumable on the existing queue, progress events consumed by the loader.

### 7.6 Multi-provider LLM connections (decision 4) — the riskiest change

- The agent loop may accept providers other than OpenRouter, **handled with the most care of anything in this brief**: provider adapters behind one interface in `packages/agent`, tool-calling semantics identical across providers (strict tool schemas, parallel tool results, streaming events), the same `AgentEvent`s, the same context accounting.
- Resolution stays in `services/llm-keys.ts`; every call site keeps using `requireApiKey` / `apiKeyOrEmpty` (extended, not bypassed).
- Its own phase (§9, Phase 4), feature-flagged per org until the eval comparison on that provider matches the OpenRouter baseline for the same model family. A provider that cannot pass the comparison ships as "Coming soon", not as a working tile.

### 7.7 Database verification (decision 6)

`db_verify` uses the saved vault connection; find/count or a single SELECT only; refuses anything else. Read-only is proven at connect time (§11.4).

### 7.8 Evals

Baseline before, re-run after every agent-touching phase (§5.8). Add: an eval story that sets up a project through the structured charter and checks a fresh chat knows scope, rules and notes without navigating (extend 088); stories for each toggle on/off; a story per enabled LLM provider mirroring 088.

## 8. Acceptance criteria

1. A new QA project can be set up from "New project" to the workspace through all six steps with no stub: every visible control works or is marked unavailable with a reason.
2. Every step-1 choice (platforms, browsers, sizes, run targets, devices, OS, environments, the four version kinds) is persisted as structured data, returned by `GET /api/projects/:id`, shown identically in the charter and in the workspace; runs are not multiplied across it.
3. Files uploaded at setup appear in Artifacts with correct roles; builds appear under `project.mobile.targets`; a first chat can `read_artifact` / `list_requirements` them.
4. Each toggle changes observable behaviour in an API test, on and off, exactly per §6.5, floors included; hint copy matches; a floor is stated under its control.
5. Notes/rules of ≥ 3,000 characters reach both `buildSystem()` and `runnerSystem()` untruncated (unit test); nothing from setup is written to memory (test asserts no `writeFact` call).
6. A fresh chat on a newly set-up project answers "what do you know about this project?" from setup data alone: 088-extended passes; stories 8, 102, 103, 104, 106, 136, 156, 272 pass at the baseline rate.
7. Every integration connects in ≤ 3 clicks after the credential is in hand and shows a truthful tested state; MongoDB / Railway MCP servers keep credentials across API restarts; two projects can register the same MCP preset; readiness finds the wizard's GitHub token; inbox prerequisites are surfaced.
8. MongoDB and Postgres connections: URI in the vault, write-capable credentials refused at save with a clear message, `db_verify` works through the saved connection and refuses non-read operations; SQL Server / MySQL / Oracle tiles are Coming soon.
9. LLM page: each enabled provider connects in ≤ 2 clicks at org scope, the agent uses it for turns, the CLI lane is reachable; providers behind the flag show Coming soon. Eval comparison recorded per provider.
10. Drafts: hidden until finished, resumed after refresh, single draft per user, deleted on cancel (with confirm after uploads), purged after 7 days; only finished projects start the setup job.
11. Setup job: loader lines map to completed work; deterministic part ≤ 60 s on the fixture set; hard cap 90 s then the workspace opens; no-key path finishes without the "analyzing" copy; killing the API mid-job and restarting resumes it.
12. Walkthrough: 6–7 steps, per user, tester workspace, first finished QA project, dismissable, reopenable from Help/Settings; not shown on `EnterpriseShell`.
13. Charter page: animated in normal mode; complete and static under reduced motion and without WebGL; first paint and bundle delta recorded.
14. `npm run typecheck`, `npm run test:api`, `npm run test:e2e` (QA) and `npm run test:e2e:loc` pass; `zz-qa-mode-unchanged.spec.ts` unchanged in intent; honesty guards pass if stories/traceability were touched; `apps/api/src/loc`, `apps/api/src/video`, `services/*` untouched (diff proves it).
15. Setup copy is en-US with global placeholders; Aadhaar/PAN/SSN remain in the masking engine.

## 9. Phased approach

Branch: `feat/product-charter` (current, on latest `dev`). Each phase ends with its checks, a short written status (what works, what was measured, what is left), and owner review before the next phase.

**Phase 0 — Plan (no code).** Confirm every line reference in §5. Produce `docs/PLAN-charter.md`: structured charter data model (shared types, draft flag, migration note for existing projects), routes/services, charter block + tool + always-on skill, toggle wiring per §6.5, DB connection design incl. the read-only probe (§11.4), MCP fixes, provider adapter design and flag (§7.6), setup job design and progress protocol, draft lifecycle, the six-step UI structure with testid plan and loc-helper updates, walkthrough steps with one-line justifications, 3D/WebGL approach with fallback, test and eval plan, risks. Take the eval baseline (§5.8) on the unchanged branch and record the numbers. Owner approves.

**Phase 1 — Data model and agent seam.** Shared charter fields + draft flag; services/routes; `charterFinish` writes structured data (and generated `conventions`); charter prompt block in both prompts, untruncated; charter tool; always-on project skill; toggles wired with floors; product-model/role seeding; unit tests; eval 088-extended + toggle stories; evals vs. baseline.
*Checks:* typecheck · test:api · evals diff · graphify update.

**Phase 2 — Setup UI (QA), six steps.** Project & Scope with full matrix and versions; Uploads page; Connections and LLM pages (UI, existing mechanisms); Rules with the field; draft resume/hide; brand marks; en-US copy; both themes; reduced motion; loc helpers updated.
*Checks:* typecheck · test:e2e QA + loc · screenshots of every step in both themes reviewed by the owner.

**Phase 3 — Connections and databases.** ≤ 3-click credential flows with provider deep links; MCP env persistence and per-project naming; readiness token fix; inbox prerequisites surfaced; MongoDB + Postgres connection kind with vault URI, read-only probe, `db_verify` fixed; Coming-soon DB tiles.
*Checks:* typecheck · test:api (new connection and DB tests) · manual connect of each integration against a real account, recorded in the status.

**Phase 4 — LLM providers (flagged).** Provider adapters in `packages/agent` behind one interface; `llm-keys.ts` extended for org-scope connections and precedence; LLM page tiles live per provider only after its eval comparison matches baseline; CLI lane surfaced.
*Checks:* typecheck · test:api · evals per provider vs. baseline · flag off by default until approved.

**Phase 5 — Setup job, loader, walkthrough.** Job kind + progress events; full-screen setup state with the §6.8 timing rules; resume on reconnect; first-time walkthrough with Help/Settings reopen.
*Checks:* typecheck · test:api (job resume, no-key path, 90 s cap) · e2e for loader and walkthrough · evals re-run (the job touches agent inputs).

**Phase 6 — Charter motion and polish.** Animated/3D charter with fallbacks; final pass on spacing, icons, copy, accessibility; animation performance; bundle and first-paint numbers.
*Checks:* full `npm test` · both e2e configs · honesty guards · final eval run vs. baseline · PR with the status table and the measured numbers.

## 10. Decision log (owner, 2026-10-18)

1. Uploads: own optional page. "Tell the agent about your platform" stays on Rules.
2. Nothing mandatory; every connection also in Settings.
3. No OAuth pacts with big providers now; normal credential flow, fewest clicks (≤ 3), provider redirect acceptable, must end verified and persisted.
4. Multiple LLM providers (Copilot, separate Claude setups named); own LLM or own key; **global scope**; agent loop may accept other providers, handled with great care.
5. CLI = surface the existing paired-desktop CLIs; nothing new.
6. Databases: MongoDB + Postgres live, direct connection with URI in the vault, tested on save; SQL Server/MySQL/Oracle Coming soon; read-only enforced by the product (write probe + limited `db_verify`); fix `db_verify` or do not offer the tiles.
7. Versions: everything, in a US-aesthetic matrix.
8. Toggles: all real, defaults on, floors stated (table in §6.5).
9. Prompt is the source of truth; structured charter fields injected every turn and run, uncut; no memory dual-write; vault names only.
10. Always-on project skill: approved (`selectSkills` flag).
11. Setup job timing and content rules (§6.8).
12. Walkthrough per user; planning session picks the 6–7 steps as senior UX.
13. Refactor the shared shell; loc behaviour unchanged; helpers updated in the same commit; no loc-only features here; no QA-only data model.
14. Draft flag with the lifecycle in §6.7.
15. Branch `feat/product-charter`.
16. WebGL and 3D allowed.
17. Walkthrough: tester workspace, first finished QA project, dismissable, reopen from Help/Settings; no tour on `EnterpriseShell`.
18. en-US copy; global placeholders; keep Aadhaar/PAN/SSN in masking; hint wording as in §6.5.
19. Save and show the full matrix; no auto-multiplied runs; cross-browser campaigns later.
20. "ethical" → aesthetic.

Added during Phase 0 planning (2026-10-18), recorded in `docs/PLAN-charter.md` §14:

21. Charter animation: SVG + `motion` + CSS 2.5D, no three.js; lazy chunk; static document under reduced motion. True 3D can replace the canvas component later.
22. Landing after setup: everyone lands in the tester workspace of the new project with the walkthrough; the session persona is set to tester (switchable back). `EnterpriseShell` gets no tour (decision 17 intact).
23. Confirmed: notes hard cap 20,000 chars; charter-less projects keep today's behaviour (`LEGACY_RULES`); extended PII masking applies to all agent-visible text; the eval API runs on a temp file store, never the Atlas cluster; setting `INBOX_SMTP_PORT` / `INBOX_DOMAIN` on the Railway `api` service is the owner's action.

Wizard v2 (owner, 2026-10-18, after the review of the six-step setup):

24. Every integration tile really connects. Native defect filing for GitLab, Azure DevOps and Bugzilla joins Jira, GitHub and Linear; every other catalog tool connects with its own API token, tested on save against its own API, and the agent uses it through `integration_request` (credential added server-side). A search miss offers "Connect any REST API", "Add an MCP server" and "Request it".
25. Logos are the official full-colour marks, bundled with the app (never fetched at runtime), on an Apple-style depth tile.
26. Anthropic, OpenAI, Azure OpenAI and Google Gemini (OpenAI-compatible endpoint) are live with the organization's own key; Amazon Bedrock and Google Vertex AI are "on request".
27. Providers go live now; the §9.6 parity evals run as soon as a direct key exists (reverses decision 4's promotion gate). Fake-provider tests prove the path end to end meanwhile.

Localization v2 and the prewarm (owner, 2026-10-18, after the localization setup review):

28. Narrow edits inside `apps/api/src/loc` are allowed at the model-call seam only (`mt.ts`, `glossary.ts`, `validate.ts`, `media.ts`, `import.ts`, `router.ts`, and the callers those signatures force), so translation, review, glossary and media AI run on the project's chosen provider. Everything else reaches the loc services through their exported functions.
29. Setup uploads route themselves: content to translate → Assets; TBX and glossary sheets → the project termbase; TMX → the translation memory; style guides and briefs → Knowledge (references); APK / AAB / IPA → Builds. Each card says where its file went and offers a one-click move.
30. The setup loader adapts: a real "Preparing your workspace" stage ends as soon as the app is warm, at most 30 s after the blocking setup steps. It never waits for nothing.
31. No pre-launch of the cloud browser or the virtual device during the loader; views, data, files and the agent's context are warmed.

## 11. Flags: where a decision meets a constraint (handle in the plan)

1. **Agent loop protocol.** `packages/agent` speaks OpenAI-compatible chat completions. Anthropic direct uses a different message/tool format; Bedrock/Vertex add auth layers. Each adapter must reproduce tool-calling, streaming and context accounting exactly, or the eval comparison will show it. Ship behind the org flag (Phase 4) and treat "Coming soon" as the honest default for any provider that is not verified.
2. **GitHub Copilot** has no key-based public completions API for third-party agent loops; it fits the CLI lane (like Claude Code / Codex on the paired desktop), not the key tile. The plan should present it that way rather than promise a key field that cannot work.
3. **"Global" LLM scope** meets a per-user key system. Implement as an org-level connection in `llm-keys.ts` with explicit precedence (project model choice → org connection → user key → operator env) and show the user which one is active.
4. **Read-only probe without writing client data.** "Probe a write, expect it to fail" must never leave a row behind on a writable account. Postgres: attempt the write inside a transaction and always `ROLLBACK` (or check `current_setting('transaction_read_only')` and role grants). MongoDB: introspect privileges (`connectionStatus` / `usersInfo` roles) rather than inserting; if an insert probe is unavoidable, use a dedicated scratch collection name and delete it in the same step, and document that behaviour under the control.
5. **Test inbox** depends on `INBOX_SMTP_PORT` / `INBOX_DOMAIN` that no deployment sets. Either set them for the Railway API service as part of Phase 3 or show the tile as needing operator setup.
6. **Local devices / desktop options** need a paired machine and the org flag; the wizard cannot complete pairing. Keep the selection, show the prerequisite, and let the setup job/workspace finish it; never silently fall back.
7. **Draft flag vs. `listProjects`.** Drafts must be filtered out of every list that clients see (web, mobile, enterprise dashboards, MCP `projects_list`) and purged by the existing scheduler leader, not a new process.
8. **Loader truthfulness vs. the 90 s cap.** The UI needs a progress contract (ordered events with the step name and completion) from the job, not a percentage guess; design it with the job in Phase 0.
9. **Loc helpers** encode the current step order (`next` ×3 then `create`). Six steps break `finishCharter`; update it with the step restructure in the same commit (decision 13).
10. **Persona after creation.** Admin/manager users land on `EnterpriseShell`; the loader must still run and finish for them, and the walkthrough is simply not shown (decision 17).
