# Localization mode

One product, two project kinds. A **localization project** re-lays the same shell — chat on the left, a 56 px rail, the
workbench tabs on the right — and colours the accent orange (`html[data-mode='localization']`). QA projects are
untouched: no `kind` is stored on them and every code path reads `projectKindOf(project) === 'qa'` (story 1114).
Stories 1000–1114 are in [loc-agent-stories.md](loc-agent-stories.md); status per story group is in
[story-matrix.md](story-matrix.md), checked by `scripts/check-story-matrix.mjs` in CI.

## Three seams

| Seam | Owns | Never imports |
| --- | --- | --- |
| `packages/loc-formats` | filters (SCORM, HTML, strings, XLIFF, Office, subtitles…), inline-code masking, sentence splitting, CLDR plurals, TMX/TBX/XLIFF, zip safety, language detection | anything from `apps/` |
| `apps/api/src/loc/` | assets + ingest, segments + QA rules, TM, terminology, routing, MT engine, estimate, deliveries + gates, dashboard, floor/governance, validation, screens, preview/player | format parsing (it calls the filters) |
| drivers (`packages/browser-driver`, `device-driver`, `desktop-driver`) | opening the localized build, screenshots, DOM facts | localization concepts |

## Content model

`LocAsset` (bytes in the object store under `loc/<project>/<asset>/`) → `LocDocument` (one per file / page / slide) →
`LocSegment` (stable key `<documentPath>#<sourceType>:<locator>`, source with `{{cN}}` masks, `codes[]`, `tags[]`,
`plural`, `linkedKey`, `translations[locale]` with `rev`, `qa[locale]` flags). Re-ingesting the same name creates the
next version and carries translations over for unchanged sources; identical bytes are refused (409).

## Jobs

`loc_ingest`, `loc_translate`, `loc_qa`, `loc_export`, `loc_validate`, `loc_media`, `loc_import` run on the same durable
queue as runs (`jobs/loc-worker.ts`): checkpoint per batch / page, pause and cancel from any device (cancel takes effect
between batches), park with a plain reason when no OpenRouter key exists, resume after a worker dies. Ingest and export
need no model. A translate job never overwrites a row a person edited after the job planned it.

**Progress** (`loc/progress.ts`): every job reports through `reportProgress` — segments per locale for translate
(`JobProgress.byLocale`), an ETA once there is a pace (`Pace`), the ids being written (`inflight`, read-only in the
Workbench until they land) and, for packages, `parts.{text,images,media}`. The `loc:progress` event drives the Assets
row, the asset panel, the Workbench badge and file tree; asset stats move after every batch (`bumpAssetStats`).

**Notifications** (`services/notifications.ts`, `routes/notifications.ts`): finished / failed / parked jobs, jobs that
finished with a **warning** (`loc.job.warning`: done, but something needs a person — no browser for the visual check,
strings left untranslated), assignment, review requested, a linguist's submitted review (`loc.task.submitted`), due
soon (scheduler sweep), @mentions and defects land in the bell (`notification:new` to the `user:<id>` room) and by
email unless the user turned it off in Profile.

**Outcomes** (`JobOutcome` = run status | `warning`): a job ends `passed`, `warning` (with `result.attention`: what is
left for a person), `failed` or `cancelled`. A job that translated nothing fails; one that translated most of the work
warns. `explainLocJob()` (shared) turns any job into a tone, a plain sentence and fix actions (open Settings → Browsers,
add a key, pick a model that hears audio, retry) — the bell, the chat card and `loc_job_status` all read it.

**Browsers** (`loc/browser-target.ts`): validate and capture pick the desktop in-app browser when a paired machine is
connected, else Browserless (falling back to the desktop when the endpoint refuses). With no browser at all, validate
runs the text checks and ends with a warning instead of failing.

## Translation engine (`loc/mt.ts`)

Batches by document (≤ 40 items / ≈ 1,800 words, plural sets never split) → memory hints + terminology constraints +
style guide in the prompt → strict JSON → per-item validation (count, ids, placeholder parity, script, echo) → one
stricter re-ask → escalation to the best tier (bypassing a pinned model) → `needs-human` flag. Provider 429/5xx fall
back within the tier. Exact and in-context (101 %) memory matches are applied without a model call. The MQM review
always runs on a model different from the translator (`route({ not })`).

## Routing (`loc/router.ts`)

`task → tier → model` after the org allowlist, the org task policy and the ZDR filter, in this order: the model chosen
for the language (`locales[l].model`, translate tasks), the project pin (`taskModels`), per-pair eval priors
(`loc_evals`), the product default (`PRODUCT_DEFAULTS`, while the tier is unchanged), then the tier shortlist. Settings →
Localization is three defaults (translation, reading text in pictures, redrawing pictures), one optional model and a
"dub" switch per language, and everything else (tiers, ZDR, gates, style, memory, voices, the full routing table)
under Advanced. `GET /loc/model-settings` and `GET /loc/models?task=` feed it.

Routes check what a model can take in: `speech.stt` only picks models whose catalog `inputs` include audio (the
`AUDIO_IN` shortlist) and is blocked with a plain reason when none is allowed. Transcription (`loc/media.ts`
`speechToCues`) goes to ElevenLabs Scribe first when a key is set (word timestamps grouped into cues), then to the
audio-capable models in order, failing over on "no endpoint supports input audio".

## Packages: everything inside is localized in place (`loc-formats/classify.ts`, `loc/package-media.ts`)

Every file in a SCORM / zip package is classified `text`, `media` or `skip` with a plain reason (vendor libraries,
minified code, the LMS runtime, styles, fonts and schemas are never translated). Text files are read by a nested filter
addressed as `entry::inner` — pages, outline, quiz, player text, Word / slides / sheets, PDF, SVG, subtitles, and the
natural-language strings inside JS / JSON / XML. Pictures and recordings become child assets (`embeddedIn`) at ingest;
one translate job reads them (OCR / transcription), translates their text with the pages, redraws the pictures and
writes subtitles (dubbing only for languages set to dub), with a priced breakdown before it starts. The export swaps
the redrawn pictures in at the same paths, adds `<stem>.<locale>.vtt` tracks and wires `<track>` into the pages,
retags translated in-place tracks, and leaves every other file byte-identical; the live preview serves the same.
The Media tab lists every file with what happens to it; any file can be included or skipped by hand (the package is
read again and nothing already translated is lost). PDFs (`filters/pdf.ts`) translate in place (source text removed
from the content stream, Noto fonts per script, overflow and missing glyphs become findings) and ship as PDF or DOCX.

## Gates and deliveries (`loc/deliveries.ts`)

One delivery per asset × locale: `translating → in-review → needs-rework → approved → delivered`. Submitting needs every
segment reviewed/accepted and no open flag; approving needs everything accepted, no critical flag, human review on
tagged content (`gates.humanReviewTags`, default legal/finance/medical), validation when required, min MQM. Approval is
an `ApprovalRequest` of kind `loc_delivery`; its final approval exports (native format through the same filter, or
XLIFF 2.1 / TMX) with a sha256 checksum.

## Validation (`loc/validate.ts`)

Opens each HTML page through the locale overlay (`…/files/~<locale>/<path>`, signed; `~source` for the original — the
language rides in the path so two panes of the same asset never share a cookie) or the live site in a real browser,
captures source and target, runs `analyzeDom` (truncation, overlap, off-canvas, untranslated text and alt, RTL
direction, `lang`, layout drift) and a vision review, records `LocScreen`s with regions for the Review tab, and marks
deliveries validated when nothing critical remains. Defects are ordinary issues with a `loc` facet.

## Image localization (`loc/media.ts`)

A picture ingests as a ready `image` asset with no segments. `loc_media` with `media.op: 'ocr'` sends it to the
`ocr.layout` route (vision model) and stores every text box as a segment (`context.region` in percent, `role` tag,
buttons/labels tagged `ui`); a re-OCR keeps translations of boxes whose text did not change. Those segments are
translated, reviewed and gated like any string. `media.op: 'image'` (OCR first if needed) then sends the source picture
and an exact replacement table to the `image.edit` route — a curated image-output shortlist (`IMAGE_OUT`: Nano Banana
Pro / GPT Image 2 on best, the flash image tiers below) called through `completeImage` (`modalities: ["image","text"]`,
non-streaming) — stores the PNG under `renders/<locale>.png`, reads it back with the vision model and flags every
translated string it cannot find (`formatting-issue`, source `vision`) while recording `media.renders[locale].coverage`.
The rendered picture is the locale's deliverable (`buildExport`) and is served at `assets/:id/render/:locale`. Every
call is booked in `images` units; no key parks the job; a painter that answers with prose fails over within the tier.

## Faithful previews (`loc/render.ts`, `loc/cards.ts`, `web/loc/viewers/`)

`renderPlanFor(name, kind)` in `packages/shared/src/render-plan.ts` is the one table the server and the viewer share:
`pdf` (Office, legacy Office, OpenDocument, RTF, Pages / Keynote / Numbers, Visio, EPS / PostScript, Illustrator),
`image` (Photoshop with its layer tree, TIFF, HEIC, camera RAW from its embedded JPEG, InDesign `.indd` page
thumbnails from XMP), `video` / `audio` (anything that is not already MP4 / WebM / MP3 / AAC / WAV / Ogg / FLAC, plus
subtitle streams inside the file extracted as WebVTT), or `null` when the browser draws the file itself. The converters
are LibreOffice headless (`SOFFICE_PATH`, serialized, one profile per call), Ghostscript (`GS_PATH`, optional),
sharp, heic-decode, ag-psd and ffmpeg; the API image installs them (`INSTALL_RENDER=1`). `renderAsset` caches each
result in object storage under `render/<sha24>/` next to the asset (`file`, `meta.json`, `poster`, `page-N`, `sub-N`),
and a translated render is made from the built translated file. Routes: `GET assets/:id/view/info?locale` (the plan,
kind, pages, layers and embedded tracks) and `GET assets/:id/view?locale&part&download` (range-aware bytes). A legacy
Office or RTF file is upgraded to its modern format at upload (`convertedFrom` keeps the original) so it can be
translated and checked like any DOCX / PPTX / XLSX. Files only their own app can draw (XD, After Effects, Premiere,
Audition, Animate, Captivate, FrameMaker, FDF / XFDF, fonts) get a file card (`GET assets/:id/card`, `/card/preview`):
what is inside, the text it holds, the preview it carries and what to export.

The viewers: `DocReader` (pdf.js `PDFViewer` with thumbnails, outline, tags, find, zoom, rotate, findings marked on
their page) for every paged file, `PictureStage` (pan / zoom, backgrounds, contrast picker, PSD layers, pages strip),
`MediaStage` (custom controls, captions per language including embedded tracks, original / dubbed audio, waveform),
the EPUB reader with reading settings, `StringsView` (subtitles with cue times, app strings, XLIFF, CSV: each entry
with its translation beside it) and `AssetCard`. `zz-loc-corpus.spec.ts` runs every file of a real, openly licensed
corpus (`.tmp/corpus/manifest.json`, not committed) through upload, preview, full size, accessibility and report.

## Jobs the user can see: one notification center (`web/NotificationBell.tsx`, `web/loc/ui/Activity.tsx`)

The bell in the left rail is the only place alerts live. It opens a full-height sheet fed by
`GET /api/notifications/activity` (running and recent jobs) and `/api/notifications`: **Running now** (progress, the
worker's message, pause / resume / cancel), **Needs you** (failed, warning, parked, assigned, due), **Earlier**; filters;
full text (nothing truncated); the job log; the fix actions from `explainLocJob`; Mark read, Dismiss and "Clear done"
(`POST /api/notifications/dismiss`). On a narrow window, or with the sidebar open, the same bell sits in the sidebar
header instead of the rail. There is no strip above the
workbench and no job toasts. `LocJobCard` is the one-line, expandable card pinned to the chat reply for a job the agent
started (`loc:job` agent event). The workspace socket is authenticated so `job:state` and `loc:*` live events reach the
views: Assets rows read `translating 12/25`, the Workbench grid refreshes once per landed batch, and the Progress column
reports translated % (draft + reviewed + accepted; accepted drawn solid).

## Review pairs, defects and team work

- **Review → Translations** (`loc/import.ts`, `loc/review.ts`): upload an original and the translations received;
  each file's language is read from its text and name and confirmed; `loc_import` lines it up by key, then order, then
  shape (pictures by box position, recordings by cue timing), writes external drafts, then rules + MQM run. Sets show
  per language the alignment, MQM and open findings, the two files side by side (running courses, pictures with their
  boxes and a draw-a-region tool, players in step, PDF pages) above the lined-up strings. The corrected file downloads
  in its own format; the report as PDF and XLSX. XLIFF targets present at ingest arrive as external translations.
  The page: a list of sets on the left (thumbnail, languages with MQM, open findings; search, a filter — with findings
  / lining up / failed —, Select to remove several, an Add screens card, a summary; collapses to thumbnails, a dropdown
  on narrow screens). The open set: one header line with ⋯ (from screenshots, add a translation, open in Assets,
  remove from Review), the languages as one bar (✕ removes a language, + adds one), the line-up counts and a Report ▾
  menu (PDF, XLSX, corrected file). Pictures sit in two frames of one height (drag the grip to resize; remembered per
  browser), each at its own shape with its own outlines (`targets[locale].layout`); Scroll together keeps both on the
  same content using the strings found on both sides as anchors. Outlines: blue lined up, orange a finding, red
  critical, dashed no counterpart; hovering one lights its pair, clicking one picks its string, picking a string
  brings both into view. Removing archives the set (the source files stay; screenshots can be attached again).
- **Review → Translations from screenshots** (`loc/capture-review.ts`): screenshots of an app, site or desktop app saved
  with `capture_screen` in several languages pair by screen name (suggestions by layout need a yes); the original
  language becomes the Original, each other language its translation, and the strings line up from the screenshots'
  element maps. New review → From screenshots, Artifacts → Screenshots → Send to Review, or the agent
  (`loc_review_from_captures`). Details in [SCREENSHOTS.md](SCREENSHOTS.md#into-localization-review-loccapture-reviewts).
- **Review → Visual check** is the screenshot validation (`loc/validate.ts`).
- **Defects** (`loc/defects.ts`, `services/tickets.ts`): a finding or a drawn region becomes an Issue with a `loc`
  facet (language, texts, crop) and, when the project has GitHub / Jira / Linear credentials (variables or vault), a
  ticket linked back.
- **Team work** (`loc/collab.ts`): delivery assignees and due dates (defaults from the language's owners), comments per
  string and language with @mentions, every revision kept and restorable.
- **Capture for review** (`loc/capture.ts`, `POST /loc/screens/capture`): a `loc_validate` job with `payload.capture`
  shoots a live URL or an asset page in the source language and each target (the target address from hreflang
  alternates, else the URL pattern), records screen pairs with the DOM checks, and — with `assignTo` — creates the task.
- **Screen review** (`web/loc/review/ScreenReview.tsx`, `loc/screens.ts`): side by side / overlay / swipe compare,
  zoom and pan, Acrobat-style marks (rectangle, ellipse, line, arrow, pen, highlighter, text box, pin) with severity,
  category and reply threads (`LocAnnotation`), "Raise defect" from a mark (the crop goes on the issue), and a submit
  outcome (approved / changes requested). Pictures in Assets open the same annotator.
- **Tasks and My work** (`loc/tasks.ts`, `LocTask`, `web/loc/MyWorkView.tsx`): a review task names screens, segments or
  deliveries, an assignee, a due date and who asked (a person or the agent). My work is a Kanban / list of everything
  assigned to me across projects with due chips; submitting notifies the requester (`loc.task.submitted`) with the
  outcome, summary and open marks. Assigning a delivery creates its task; approving or delivering closes it.

## Setup: the localization brief (`services/loc-charter.ts`, `agent/loc-charter-block.ts`)

A localization project is set up in the same seven pages as a testing one (Project, Languages, Uploads, Connections,
LLM, Brief, Review) and saves a structured charter with its own `localization` section (`LocCharterSection` in
`packages/shared/src/charter.ts`): source, targets, a default tone and a register per language, content types and
their formats, where the content lives (website, app with its store link, or files), the five guardrails and the
brand terms. The brief's eight questions (`LOC_CHARTER_QUESTIONS`) are answered verbatim.

What the brief really changes, applied by the setup job (`jobs/setup-worker.ts` `runLocSetup`, steps `LOC_SETUP_STEPS`):

| Setting | From | Read by |
|---|---|---|
| `localization.locales[t].formality` | tone + per-language register | `mt.ts` `styleBlock` |
| `localization.gates.humanReviewTags` | "A linguist reviews legal, medical and financial text" (on: legal, finance, medical; off: none) | `deliveries.ts`, `segments.ts` |
| `localization.gates.blockExportOnCritical` | "Block delivery on critical issues" | `deliveries.ts`, `segments.ts` |
| `localization.dnt` | the brand question's lines, when "Keep brand names untranslated" is on | `termbase.ts` `termsForProject` → MT prompt, QA checks |
| "Approved terms (from setup)" termbase | `term → translation (locale)` lines | MT terminology block, QA checks |
| weekly accessibility check + gate | "Check accessibility of every delivery" | `a11y/schedule.ts` |
| `conventions` (`renderLocConventions`) | languages, content, rules, the brief verbatim | every translation prompt, the agent |
| `localization-brief` skill (always on) | where the content lives, uploads by destination, guardrails, the in-context procedure | the agent every turn |

Setup uploads route by kind (decision 29): content to translate becomes an asset (ingested, then summarised),
glossaries and memories are imported, style guides become Knowledge references, builds go to Builds. Every ingested
asset gets the agent's two- to three-line summary (`services/loc-asset-gist.ts`, the shared summary queue, the
project's own provider; a structure-based summary without a key). The agent reads the brief (`locCharterBlock`) and
one line per asset with its counts, progress and summary (`locAssetsBlock`) every turn; `get_project_charter`,
`loc_list_assets` and `loc_asset` carry the same data.

Model calls (decision 28): every loc model call resolves the project's provider through `locLlm()` in
`services/llm-keys.ts`. OpenRouter behaves exactly as before; a direct Anthropic / OpenAI / Azure / Gemini key maps
each routed slug to that provider's own id or falls back to the connection's lead (quality tasks) or step model.
Image editing and generation, video generation, speech models and TM embeddings still need an OpenRouter key and say
so by name; vision tasks need a vision-capable model.

## What the agent can do (`agent/tools/loc.ts`, `skills/loc-lead/SKILL.md`)

Anything the UI does, from chat. The skill tells the agent to ask (`ask_user`) when a person, language, asset or scope
is ambiguous, and to do the real action rather than write a document about it.

- **Bring content in:** chat attachments (`loc_ingest_artifact`), a URL or site (`loc_ingest_url`), a vendor's
  translation (`loc_stage_target` → `loc_import_target`), glossaries (`loc_termbase_import`), memory (`loc_tm_import`).
- **Translate and check:** `loc_estimate`, `loc_translate`, `loc_set_translation`, `loc_bulk` (accept, review, reject,
  clear, propagate, retranslate), `loc_back_translate`, `loc_consistency_check`, `loc_qa`, terminology and memory search.
- **Pictures, video, audio:** `loc_localize_image` (read text boxes, redraw), `loc_media` (transcribe, subtitles, dub).
  Words painted into a video's picture are not handled here: a video has a "Translate this video" button that copies
  it into Video Translate.
- **In context:** `loc_validate`, `scorm_walk`, `loc_capture_pair` (EN + target screenshots, optionally assigned).
- **Screens of an app in several languages:** the device / browser / desktop tools with `capture_screen`, one language
  per pass (`list_captures coverage=true` between passes), then `loc_review_from_captures` — the workflow is in
  `skills/loc-app-screenshots/SKILL.md`.
- **People:** `loc_team` (resolves a name; two matches means ask), `loc_assign`, `loc_task_status`,
  `loc_review_feedback`, `loc_upload_screen`, `loc_annotate_screen`, `loc_comment`.
- **Deliver:** `loc_delivery_status`, `loc_submit_for_review`, `loc_transition_delivery` (never to approved),
  `loc_request_delivery_approval`, `loc_export`.
- **Settings:** `loc_update_settings` (languages, dubbing), `loc_set_models`, `loc_routing`.

A localization chat opens with starter prompts and a "What can Studio do?" panel listing the same capabilities.

## The Workbench rail (`web/loc/workbench/PageGlance.tsx`)

- **Page glance**: the page in the original or the translation (flags on the toggle).
- **Contents** (`ContentsOutline.tsx`): the file as Word's navigation pane shows it — title and headings as a foldable
  tree, the text under each heading folded into one line ("3 paragraphs · 1 button") that opens, an icon per kind of
  block (`segmentType` reads the element from the locator: `h2[3]`, `li[0]`, `img@alt[1]`), a translated dot, search,
  "Show all text", and ↑ ↓ ← → keys.
- **Pictures** (`PagePictures.tsx`): thumbnails of the pictures this page shows (read from their `<img>` tags into
  `imageFlags[].pages`), each once, with its state per language; a click opens `ImageLightbox` — original beside the
  redrawn picture (or either alone), zoom, ← → through the page's pictures, Mark up, Edit the text in it, Open in
  Assets.

## Where things surface

- Tester: Assets (with the course running in each asset's Preview) · Workbench (with "Run course") · Review
  (Translations · Visual check) · Glossary & TM · Deliveries · My work · Dashboard tabs; Settings → Localization. The
  Player tab is gone; its links open the Workbench on "Run course".
- Manager: Localization floor (`/api/orgs/:orgId/loc/floor`), mode switch All · Testing · Localization (`?kind=` on
  every enterprise poll through `allowProjects(scope, ids, kind)`).
- Admin: Localization governance (`/api/orgs/:orgId/loc/governance`, writes `OrgSettings.localization`).
- Agent: `loc_*` tools (`agent/tools/loc.ts`), skills `loc-lead`, `translate-content`, `terminology-management`,
  `lqa-review`, `loc-delivery`; campaign items of kind `localization` priced by `loc/estimate.ts`.
- CI / MCP: `teststudios loc push|pull|status|validate` over `/api/ci/:project/loc/*`; MCP `loc_assets_list`,
  `loc_search`, `loc_status`, `loc_translate`, `loc_deliveries`.

## Not built yet (honest list)

Delivery pushes to CMS / LMS / Figma / Phrase (defect tickets to GitHub / Jira / Linear are built), eval A/B runner UI,
estimate-accuracy tile, locale-scan monitor, global Search facets for segments, CJK glyphs in translated PDFs without
`LOC_PDF_FONT_DIR/cjk.ttf`. Each is marked `partial` or `deferred` in the matrix.

## Tests

- `npm run test:formats` — filters, round-trips, zip safety, plurals, interchange, package classification and nested
  filters, PDF / SVG.
- `npm run test:api` — every `apps/api/test/loc-*.test.ts`, against the scripted OpenRouter (`test/fake-openrouter.ts`)
  and fake ElevenLabs / trackers (`test/fake-services.ts`): release A (progress, routing, notifications), release B
  (package children, fan-out, export, preview, PDF), release C (import per file type, reports, defects, team work),
  UX repair (`loc-ux-repair.test.ts`: preview pinning, audio routing and fallover, Scribe, warning outcomes, pictures
  auto-translated before render, validate without a browser, tasks, annotations) and agent parity
  (`loc-agent-tools.test.ts`: attachments to assets, ambiguous names, assign and read feedback, settings, models,
  glossary import, no self-approval, capture without a browser).
- `node --import tsx --test apps/api/test/render.test.ts` — the render plan, camera RAW and InDesign previews, the
  Photoshop layer tree, LibreOffice conversion (skipped without LibreOffice), file cards and embedded caption tracks.
- `npx playwright test -c apps/web/e2e/playwright.loc.config.ts zz-loc-corpus` — the real-file journey (set
  `CORPUS_ONLY` to a regex of file names; screenshots and `steps.jsonl` land in `.tmp/corpus-journey/`).
- `npx playwright test -c apps/web/e2e/playwright.loc.config.ts zz-loc-ui-audit` — the UI audit: every localization
  view at 1440 / 1024 / 768 px in light and dark with the chat open and closed; axe checks plus truncation, overlap,
  unreadable text, transparent overlays and raw ids. The gallery and findings land in `.tmp/ui-audit/`.
- `npm run test:e2e:loc` (`apps/web/e2e/playwright.loc.config.ts`) — every `zz-loc-*` spec and the journeys with real
  jobs: the fake OpenRouter server on 8098, fake services on 8099, a local Browserless shim on 3005, the mail sink at
  `/api/__test/mail`. Journeys: strings, settings, failures, regressions, course (SCORM + PDF), review (text,
  screenshot, SCORM and video pairs, team work). Fixtures are in `fixtures/loc/` (see its README).
