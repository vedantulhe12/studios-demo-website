# KaDeep Studio Pro 2.0.75 — development build (`build-75`)

Desktop installers use version `2.0.75` and talk to the development server (grip.kadeep.ai). Product version stays `2.0.0`.

## Accessibility reports clients can read

- **Reports tab** under each asset's Accessibility tab. The default report is an A4 document in the KaDeep Studio style: cover, a score and verdict infographic, a violation matrix by WCAG principle and severity, every finding with its page and fix, and a confidence summary. It downloads as a tagged PDF with an outline, or as HTML.
- **Custom reports from the chat.** Ask the agent for a report ("an executive summary for the client", "only the serious issues, by page") and it builds one with the same branding.
- **Confidence tab.** How far to trust a result: a confidence score per check, a seeded-defect benchmark (35 automated planted defects across PDF, Office, EPUB, HTML, pictures, media and InDesign, plus manual seeds), and a sample for a person to review.
- **Honest verdicts.** A check the tool could not run on a file now says **Not evaluated** instead of passing it. Counts read correctly ("1 page checked", "12 stories checked").

## InDesign

- IDML and ICML files are read: stories, styles, pictures with their alt text, language tags and reading order. They can be translated, checked for accessibility and previewed as a Story view.
- A binary `.indd` shows the page previews stored inside it, and explains how to export IDML to translate it.

## Real previews for every file

Every asset now opens in a viewer that draws it the way its own app does.

- **Document reader** (pdf.js, the reader inside Firefox): real pages with selectable, searchable text and form fields; thumbnails, outline and tags panels; page number, zoom presets, find with match counts, rotate, download and full screen. Accessibility findings are numbered on the page they belong to.
- **Word, PowerPoint and Excel**, legacy DOC / PPT / XLS, OpenDocument, RTF and Pages are converted on the server with LibreOffice, so fonts, pictures, charts and layout look as they do in Office. **Layout** shows the faithful page, **Structure** pins findings to pictures, tables and cells. A legacy file is upgraded to its modern format on upload so it can be translated and checked too.
- **Pictures and design files**: pan and zoom, transparency backgrounds and a WCAG contrast picker. Photoshop (PSD) shows its composite and a layers panel with the text inside text layers. HEIC, TIFF, camera RAW (DNG, CR2, NEF, ARW…), EPS and Illustrator are supported. PDF-compatible Illustrator files can be translated and checked like PDFs.
- **Media player**: scrubbing with hover time, frame step, speed, captions per language (including subtitle tracks inside an MKV or MP4), Original / Dubbed audio, loop, picture-in-picture, full screen and keyboard shortcuts. MOV, MKV, AVI, WMV and other formats are converted for the browser. Audio gets a waveform. The accessibility check counts subtitle tracks inside the file as captions.
- **E-book reader**: table of contents, chapters in reading order, and reading settings (text size, white / sepia / night, line length, typeface), remembered per reader.
- **Subtitles and strings files** (SRT, VTT, app strings, XLIFF, CSV) show every entry in order, subtitles with their cue times, with the translation in the chosen language beside each one and a search box.
- **Adobe project files** that only their own app can draw (XD, After Effects, Premiere Pro, Audition, Animate, Captivate, FrameMaker, Acrobat form data, fonts) get a file card: what the file is, what is inside it, the text it holds, the preview it carries, and what to export to translate and check it.
- **Original and translation side by side** works for every one of these viewers.

Tested on 69 real, openly licensed files: government reports and forms, decks, workbooks, Adobe files, photos, video, audio and e-books.

## For operators

- The API image installs LibreOffice, Ghostscript and Noto fonts for the converters (`INSTALL_RENDER=1`, the default; set it to `0` for a slim image). Converted previews are cached in object storage next to the asset.
- Optional settings: `SOFFICE_PATH`, `GS_PATH` and `FFMPEG_PATH` point at the tools when they are not on `PATH`.
- Files with hundreds of documents (a large IDML, a big site) now ingest in seconds: documents and segments are written in batches and progress is reported a few times a second.

---

# KaDeep Studio Pro 2.0.74 — development build (`build-74`)

Desktop installers use version `2.0.74` and talk to the development server (grip.kadeep.ai). Product version stays `2.0.0`.

## Many agents on one request

Big asks no longer run on one agent, one step at a time. "Write 50 test cases for the checkout", "run this suite on Android", "hunt bugs across these modules" or "translate this into twelve languages" is split into separate pieces, and many agents work on them at once.

- **Agents in the composer.** Pick how many agents may work at once (Auto, or 1 to 64) next to the effort control.
- **A live card in the chat.** It shows the plan, then each agent at work: what it is doing, its live screen, and how long it has run. There is a speed-up meter (how much faster than one agent), time left, cost so far, and pause, resume, cancel and agent-count controls. Click an agent to read its transcript.
- **Cost first.** You see an estimate before anything starts. A big job waits for **Start** on the card, and a running swarm pauses if it costs half again more than estimated.
- **Exactly what you asked for.** Parallel agents never write the same case twice, near-duplicates are labelled, and an agent that stops short gets a second round, so "write 50" yields 50.
- **Runs that hold up.** Each running case gets its own browser or device, and its own test account when the project has several (`TEST_EMAIL_2`, `TEST_PASSWORD_2`, …). Mobile devices stay booted between cases and the app is restarted, so a 200-case run does not boot 200 devices. Desktop agents take turns on the paired machine.
- **Failures checked twice.** A failed case is re-run by an independent verifier agent. Bugs from the same root cause are grouped, and every one keeps its own evidence.
- **It reports back.** The agent answers from the swarm's report in the same reply. If the job outlasts the reply, the agent posts its summary in the chat when the job finishes.
- **The Runs page** lists swarms; open one to see every piece of work as a grid.

## Faster everywhere

- The agent reads and checks several things at once instead of one after another.
- Localization translation and review run six batches at a time.
- Measured: 50 test cases in 6.7 s with 8 agents against 32.6 s with one, and 1,000 cases in 31 s with 32 agents, using a scripted model. With real models, 20 cases took 22.6 s with 8 agents against 98.6 s with one.

## For operators

- Heavy runs spread across worker replicas. `GET /api/platform/queue` shows the demand, and an optional autoscaler can set Railway worker and Browserless replicas. It is off by default, and logs only until `AUTOSCALE_APPLY=1`.
- New settings are listed in `.env.example` and `docs/SCALING.md`.

---

# KaDeep Studios 2.0.0 (`v2.0.0`)

Installer filenames use product version `2.0.0`. The GitHub release tag is `v2.0.0`. Mobile build 7 (iOS `buildNumber` 7, Android `versionCode` 7).

KaDeep Studios is now two products in one: the testing studio you know, and **Localization Studio** — a workspace where your team and the agent take courses, app strings, websites, documents, pictures and video into other languages, check them in context, hand them to reviewers and ship them in their original format. Choose **Localize something** when you create a project. Testing projects are unchanged.

## Bring anything in

- SCORM 1.2 and 2004 courses and zip packages; websites by address, with each language's page found from the site's own language links; strings files (JSON, ARB, YAML, PO, properties, RESX, iOS strings, stringsdict and xcstrings, Android XML); XLIFF 1.2 and 2.1; Word, PowerPoint, Excel, Markdown and PDF; SRT and VTT subtitles; pictures, audio and video.
- Every file inside a package is accounted for — translated, localized as a picture or recording, or left alone with the reason — and any of them can be included or skipped by hand.

## Translate with your terminology and memory

- A priced estimate first, then one job that does it all: the text, the pictures (text read off, translated and drawn again per language) and the recordings (transcribed, subtitled and, for the languages you choose, dubbed with ElevenLabs).
- Translation memory and glossaries, with TBX, CSV, XLSX and TMX import; do-not-translate and forbidden terms; a model per language; a second model reviews quality (MQM).
- Live progress per language with an estimate of time left, and pause, resume and cancel from any device.
- A job that leaves some work for a person finishes as **Needs review** instead of failing, and every failure says what happened in plain words with the fix beside it.

## Check it in context

- The Workbench: every string as a card or a table row, a live preview in the original or the translation, **Run course** (the real SCORM course, scaled to fit, side by side with the original, with its LMS reporting), a Word-style outline of the page, and the page's pictures with a full-size compare view.
- Validation in a real browser — the desktop app's own browser or Browserless — marks cut-off labels, overlaps, untranslated text and right-to-left problems on side-by-side screens.

## Review with people

- **Review translations done elsewhere:** drop in the original and the vendor's files; each language is read, lined up, checked and scored, with side-by-side pairs for strings, screens, courses and video, a corrected file to download and a PDF or Excel report.
- **Capture for review:** screenshot a page in English and Korean (or any languages) and hand it to a linguist in one step.
- **My work:** a board of everything assigned to you across projects, with due dates.
- **Screen review:** side by side, swipe or overlay, zoom, and Acrobat-style marks (box, ellipse, line, arrow, pen, highlighter, text and pins) with comments, severity and category. Raise a defect from a mark — the crop goes with it, and it opens a GitHub, Jira or Linear ticket when the project is connected — then approve or request changes, and whoever asked is told.
- **Deliveries** per language move through review and approval to an export in the original format, XLIFF or TMX, with a checksum. Approval is always a person's decision.

## One notification center

- Everything lands under the bell: running jobs, what needs you, assignments, reviews and mentions — in full, with the job log, fix actions, dismiss and clear. An email copy follows unless you turn it off in Profile.

## The agent does what the UI does

- In a localization chat the agent turns attachments into assets, captures pages, finds teammates (and asks which one when two people share a name), assigns work, reads a reviewer's feedback back to you, adds languages, picks models, and imports glossaries, memories and vendor translations. When a request is unclear it asks before it acts.
- New chats open with starter prompts and a **What can Studio do?** panel.

## For managers and admins

- A **Localization floor** (throughput, queue and quality per language) and organization-wide localization governance; the dashboards switch between All, Testing and Localization.

## Design and accessibility

- An orange localization theme whose filled buttons pass WCAG AA contrast, an 11px minimum for text, and layouts that adapt to narrow panes and windows. On a narrow window the sidebar carries the notification bell, and Settings lists its sections across the top.

## Under the hood

- API: `/api/projects/:id/loc/*`, `/api/loc/tasks/mine`, `/api/notifications/activity` and `/api/notifications/dismiss`; new task and annotation collections, indexed at startup. MCP localization tools and `teststudios loc push | pull | status | validate` for CI.
- Desktop: localization links open the right page, including on a cold start on macOS. The 1.1.2 pairing fixes are included.
- Dubbing works on free ElevenLabs plans (the account's own voice is used when the built-in one is refused).
- Tests: format filters, API suites, browser journeys that run real jobs against scripted models, a UI audit (accessibility and layout checks across screen sizes and themes) and opt-in live model evals (`docs/LOC_LIVE_EVALS.md`).

---

# KaDeep Studios 1.1.2 (`v1.1.2`)

Installer filenames use product version `1.1.2`. The GitHub release tag is `v1.1.2`. Mobile build 6 (iOS `buildNumber` 6, Android `versionCode` 6).

Desktop pairing used to show three disagreeing answers at once — the app's own state, the database heartbeat and the live socket — so the pane could say "1 online" while the agent replied "no desktop worker is connected", and a two-second network blip left a red *Error: websocket error* next to a green card. This release gives each layer one source of truth.

## Worker status tells the truth

- **Offline means offline.** A disconnected machine is marked offline immediately and stays offline; a recent heartbeat no longer resurrects it for 90 seconds. `pickWorker` only hands the agent a machine whose socket is actually in the room, and the "not connected" message now distinguishes *reconnecting* from *never paired*.
- **Real machine names.** `Vedants-MacBook-Air.local · macos` is now `Vedants-MacBook-Air · macOS 15.6 (arm64)`, from one shared `describeWorker()` used by `desktop_start`, `desktop_status` and the remote driver. The app sends the product OS version instead of the Darwin kernel number, and `.local` is stripped before anything is stored.
- **Honest health.** When the first driver report has not landed, `desktop_start` says what it does know ("Health: driver reachable, Accessibility not granted") instead of `Health: unknown`.

## Pairing that cannot lie to you

- **One status line in the Desktop pane**, in priority order: terminal error, pairing progress, paired, paired elsewhere, idle. A transient socket error can no longer overwrite the progress line, and anything stale clears the moment the machine comes online or you switch project.
- **Phases in the app**: `unpaired · pairing · connecting(attempt n) · online · busy · paused · offline · error`, where `error` is terminal only (credential rejected, code invalid or used, env-override mismatch). Transport failures read "Reconnecting to KaDeep… attempt 3"; Socket.IO reasons like `transport close` become plain English. The wizard hides the paste box while a code is being redeemed, and the tray label comes from the same table.
- **Pair this computer** waits as long as the token lives instead of giving up after a minute, cannot mint a second code while the first is in flight, and resolves when the machine actually registers (the button reads *Pairing…* until then).
- **Pair this computer is no longer shareable.** `POST /workers/pair/self` returns a token bound to that machine's hostname and OS, valid two minutes, with no deep link; redeeming it from another computer is refused. The lead's 15-minute shareable code is unchanged.
- **Forget pairing really forgets.** The app tells the API (`worker:forget`), so the row disappears from the project instead of lingering as an offline ghost, and the local status drops its worker and project id.
- **One machine, one project.** Pairing a computer that belonged to another project moves it: the old row is deleted, that project's default worker is cleared, the move is audited, and the app says "Moved from Payments".
- **The desktop app starts its worker before the window**, and a pair click that arrives first is queued — first-run clicks are no longer silent no-ops. With `TESTSTUDIOS_API_URL` set, that API's credential stays in memory and is never written into `worker.json`.

## The permission gate stops looping

- A launch blocked by macOS permissions now returns an actionable error instead of a generic failure, and `desktop_hand_to_human` for a permissions reason polls the machine every 5 seconds and resumes by itself the moment Accessibility and Screen Recording are granted (capped at 5 minutes, not 20). The overlay says exactly which two toggles to flip, and the app's permission cache is invalidated after the grant flow.

## Roles

- Remove, Set default and Remove offline are hidden from testers on the web instead of failing with 403 at the API; the machine count reads "ready" and "paused" separately. The phone's worker screen tells a tester how to pair from the computer instead of showing a dead button.

## CI

- The Android APK job pins `android-actions/setup-android` to `packages: platform-tools`. Its default list still contains the `tools` package Google removed, which made `sdkmanager` exit 1 before Gradle ever ran.

The Graphify map is machine-local (`graphify-out/` is gitignored). There is no pull-request freshness check.

---

# KaDeep Studios 1.1.1 (`v1.1.1`)

Installer filenames use product version `1.1.1`. The GitHub release tag is `v1.1.1`. Mobile build 5 (iOS `buildNumber` 5, Android `versionCode` 5).

- **Pair this computer from Settings → Providers.** A fresh install of the desktop app used to land on "No computer paired — install the desktop app and pair it from the Desktop tab". Inside the app the page now says this computer is not paired yet and pairs it in one click (the Desktop tab keeps its own button); the provider list reads "Pair this computer" instead of "Needs the desktop app"; the model picker's reason line matches. On the web the copy points at the app and opens the Desktop tab.

---

# KaDeep Studios 1.1.0 (`v1.1.0`)

Installer filenames use product version `1.1.0`. The GitHub release tag is `v1.1.0`. Mobile build 4 (iOS `buildNumber` 4, Android `versionCode` 4).

## Shell and navigation

- **Account menu.** The whole profile row in the sidebar (avatar, name, dots) opens one account menu. The menu and its **Appearance** flyout render at body level, so the flyout is never clipped behind the chat pane.
- **Profile and Connections live in Settings.** Profile & account is the first Settings tab (Account group); Connections is Settings → Connect. Both left-nav entries and the workspace-panel duplicates are gone; deep links and agent `tab` events land in Settings.
- **Windows and Linux builds** no longer reserve the macOS traffic-light gutter. Only macOS outside fullscreen gets the inset (`body.traffic-lights`). Breadcrumb levels shrink to ellipses at every width instead of pushing controls off the bar.
- **Dashboard tabs** collapse into one dropdown when the pane is narrow and return to pills when it is wide.
- **Tools & skills** folders start closed.

## Stability

- **No more white screens.** Error boundaries at the app root, per pane, per workspace panel and per Runs strip show a fallback in place ("Try again") instead of blanking the window. Run records missing their arrays render as zero steps.
- **Runs never stay live after their driver is gone.** Stop, an error or a dropped CLI closes the chat's run; a failed or cancelled job closes the runs it tagged; a reaper closes silent active runs on boot and every scheduler tick. The Runs tab shows *live* only with recent activity, *interrupted* otherwise, and polls while anything is active.

## Composer and chat

- **Effort** is its own dropdown beside the model, on web and mobile. Levels come from the model: a CLI's reported efforts, or the OpenRouter scale when the catalog says the model supports reasoning. Hidden otherwise.
- **Context meter for every CLI.** Codex, Claude Code, OpenCode and Cursor all move the ring; the window is estimated from a model table when the CLI does not report one.
- **Agent marks in the chat list.** Chats that ran on Cursor, Codex, Claude Code or OpenCode show the agent's mark; several agents show a `+`; hovering opens a card with providers, models, fork lineage, dates and message count, and a Fork action.

## Desktop method for coding-agent CLIs

- A CLI chat has a **Desktop method** beside Effort: **Auto** (default — the agent picks per task), **CUA** (recorded `desktop_*` session only) or **Native** (the CLI's own shell, AppleScript / PowerShell and screenshots; the recorded driver is off for that chat).
- **`attach_desktop_evidence`** turns a screenshot the CLI took itself into a recorded step with an image on the run and a live frame, so native checks count as evidence.

## Approvals, evidence and theme

- One **ApprovalCard** frame for HITL prompts, plans, campaigns, budget checkpoints and the approvals inbox. The in-chat card now handles the `approval` kind (Approve / Deny; destructive confirmations read as such).
- Run evidence fits its pane; the manager run drawer scrolls instead of growing; strips above the runs table never clip an open campaign card.
- Manager and admin surfaces use the shell's 8px control language and icons; the last hard-coded greens and the light artifact background are gone.

## Emails

- Every email the platform sends — invites, reminders, role changes, offboarding, completion notices, approval requests (with Approve / Reject / Review buttons), SMTP drafts and the SMTP test — renders through one branded layout with the KS mark, the product accent and the legal footer. Email-change confirmations go out from the platform mailer first.

## Mobile

- 44pt tap targets and spoken labels on buttons, chips and list rows; pull-to-refresh spinner only for the pull; Effort chip beside the model; agent marks on chats; EAS preview/production builds point at the production API.

## Installers

Built by **Build artifacts** (`.github/workflows/build-artifacts.yml`). How to start a run and where files land: [BUILDS.md](BUILDS.md).

---

# KaDeep Studios 1.0.0 (`v1.0.0-beta.1`)

Public beta. Installer filenames use product version `1.0.0`. The GitHub release tag is `v1.0.0-beta.1`.

## What you get

- **Watch live (phone).** View-only live of the browser, Appetize, or paired desktop from the mobile app. The phone does not drive the session.
- **Untitled chats.** A new chat stays untitled until the first send. Pairing URLs stay off tester-facing surfaces.
- **Org dashboards.** Scope a dashboard to this project, all of your projects, or a selected set. The selected-set picker is a dialog (checkbox + name), not a wrapping chip row.
- **Compare projects tab.** Readiness, pass rate, open defects, and runs sit on their own tab. Release, Predictive, Productivity, and Bug Analysis no longer repeat that table.
- **Mobile sign-in.** The app talks to the compiled API. There is no server picker on the login screen.
- **People.** Home and People no longer share a members cache that crashed the People screen (`teams.map`).
- **Brand.** Sidebar, web login, empty chat, and mobile login show plain **KaDeep Studios** text. No logo lockup and no Pro pill on those surfaces. Web login still has the “A KaDeep product” capsule.
- **Overflow.** Dashboard header stacks (tabs, then controls). Chat titles and the live browser URL ellipsize instead of wrapping the chrome.

## Installers

Built by **Build artifacts** (`.github/workflows/build-artifacts.yml`). How to start a run and where files land: [BUILDS.md](BUILDS.md).
