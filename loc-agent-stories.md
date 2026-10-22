# Localization Mode — User Stories

*Continues the backlog. Stories 1000–1114 (1000–1099 tester altitude; 1100–1114 manager and admin altitudes, separate access, and legacy safety). Companion to the UI stories (439–487), the enterprise layer (376–438) and the integrations/video stories (504–556). Written against the plan "Localization Mode Unification": one product, two project kinds (`qa` | `localization`), one agent runtime. Each `Pass:` is observable on screen, in an export, or in a job record; each `Edge:` names the awkward case the story must still survive.*

**The model these are written against.** A project created with kind `localization` re-lays the same shell: the left nav collapses to a rail, chat stays in the middle (visually on the left), and the right pane becomes the localization workbench with its own tab strip — **Assets · Workbench · Review · Player · Glossary & TM · Deliveries · Dashboard**, plus the existing **Browser · Mobile · Desktop** live tabs for validation. Every content type (SCORM, website, strings files, XLIFF, DOCX/PPTX/XLSX, SRT/VTT, images, video, audio, Figma/PSD) becomes the same thing: an **asset → documents → segments** with protected inline codes. Long work runs as durable **jobs** (`loc_ingest`, `loc_translate`, `loc_qa`, `loc_export`, `loc_validate`, `loc_media`) that show up as progress cards in chat and in the Jobs strip. Humans act through the same `ApprovalCard` family used for QA: HITL questions, estimates, campaigns, deliveries. Localization defects are ordinary `issues` with a `loc` facet.

**Format.** Each story has: the story line with priority, `UI:` (which pane, which component, which interaction), `Agent/model:` (tool, job, sub-agent, and how the model is chosen), `Pass:` and `Edge:`.

**Sections.** LA shell & orange mode · LB asset intake · LC terminology, TM & style · LD model routing (OpenRouter) · LE agent translation · LF workbench (CAT) · LG linguistic QA · LH visual & cross-surface validation · LI media & documents · LJ human-in-the-loop, approvals, deliveries · LK CLI, MCP, CI & connectors · LL analytics · LM governance & security · LN failure modes · LO manager & admin altitudes, separate access, legacy safety.

**Altitudes.** Stories 1000–1099 are written at the tester altitude (the three-pane shell re-laid as chat + workbench). Stories 1100–1114 add the other two altitudes without a second shell: the enterprise rail gets a mode filter (`All · Testing · Localization`) applied once in `scopeFor`, managers get a **Localization floor** page, admins get a **Localization governance** page and an **Organization → Policies** screen, deliveries are approved from Mission control and the phone, and role presets (Linguist, Reviewer, Localization manager) plus deep links give people who only do localization a direct way in. Story 1114 is the legacy guarantee: QA-only orgs and projects without a `kind` see nothing change.

---

## Model routing reference (used by LD and referenced everywhere)

The catalog is discovered live from OpenRouter (`/api/v1/models`, `/api/v1/images/models`, `/api/v1/videos/models`, filtered by `input_modalities` / `output_modalities` / `supported_parameters`). Nothing below is hardcoded; these are the **task types**, the capability each needs, and the kind of model the router prefers by default. Example slugs are illustrative at time of writing; the "Recommended" badge in Settings → Models is computed (see 1024).

| Task type | Needs | Default tier logic | Example class (illustrative) |
| --- | --- | --- | --- |
| `translate.ui` (short UI strings) | text, structured JSON output, strong multilingual | Balanced tier; escalate on QA fail | Sonnet / GPT-5-class / Gemini Pro-class |
| `translate.long` (docs, courses, marketing body) | text, 128k+ context, style adherence | Best tier for `legal`/`finance`/`medical` tags, Balanced otherwise | Frontier reasoning models |
| `translate.lowres` (Indic, SEA, African, low-resource pairs) | proven multilingual quality per pair | Best tier by default; per-pair priors from org A/B (1027) | Frontier models with high multilingual eval |
| `transcreate` (taglines, CTAs) | creativity, variants, back-translation | Best tier, temperature higher, 3 variants | Frontier models |
| `postedit.fuzzy` (repair 85–94% TM matches) | cheap, precise | Economy tier | Flash / mini / open models |
| `review.mqm` (quality review) | judgment, reasons, severities | Best tier, **must differ from the translator model** (no self-grading) | A different frontier model |
| `backtranslate` | literal fidelity | Balanced tier | Any strong text model |
| `vision.lqa` (screenshot review, truncation, RTL) | image input, high-res, structured boxes | Multimodal Best tier; fallback Flash-class multimodal | Gemini Pro / GPT-5-class vision / Sonnet vision |
| `ocr.layout` (text boxes + style from images/frames) | image input, structured output | Multimodal Balanced; verified by a second pass | Gemini Flash-class |
| `image.edit` (replace text in art, keep style) | image output, `input_references`, good text rendering | Image model supporting references; verified by `ocr.layout` readback | Seedream / GPT-image / Gemini image class |
| `image.generate` (new localized art) | image output | Same as above, explicit approval | same |
| `video.generate` (re-shoot / extend) | video output, `frame_images` | Only on explicit approval, budget-gated | Veo / Seedance class |
| `speech.tts` (dubbing, narration) | audio output, voices per locale | Per-locale voice table; length-fit retries | gpt-4o-mini-tts class via `/audio/speech` |
| `speech.stt` (transcribe with timestamps) | audio/video input, timestamps, diarization | Multimodal Balanced | Gemini / GPT-4o-transcribe class |
| `embed.tm` (semantic TM) | multilingual embeddings | Fixed per TM scope (changing it re-embeds) | `AI_MODEL_EMBED` multilingual |
| `glossary.generate`, `report.compose`, `summarize` | text, JSON | Balanced | any |
| `engineering` (externalize strings, i18n wiring, PRs) | code agent in the repo workspace | CLI agent providers (Claude Code / Cursor / Codex / OpenCode) at $0 via BYO subscription | `packages/cli-agents` |

Tiers (`economy` / `balanced` / `best`) are a project setting with per-task and per-locale overrides; org `allowedModels` and the data-policy filter (ZDR-only providers) are applied before ranking. Every segment, screen, flag and generated file records `model`, `provider`, `promptVersion`, `generationId`, and cost.

---

## Journey index

Three journeys from the design narrative, as story sequences:

- **E-learning (SCORM) to hi/mr/ta:** 1000 → 1001 → 1006 → 1008 → 1016 → 1017 → 1030 → 1031 → 1049 → 1050 → 1040 → 1042 → 1045 → 1062 → 1064 → 1066 → 1068 → 1071 → 1074 → 1075 → 1076 → 1079 → 1086.
- **Fintech app to ar/de/ja with a CI gate:** 1000 → 1083 → 1081 → 1009 → 1035 → 1034 → 1041 → 1043 → 1046 → 1058 → 1059 → 1064 → 1066 → 1088 → 1082 → 1079 → 1080.
- **Launch campaign in five markets (web, images, video):** 1000 → 1011 → 1012 → 1030 → 1036 → 1068 → 1069 → 1070 → 1071 → 1061 → 1063 → 1079 → 1087 → 1090 → 1038.
- **Manager and admin running the program (separate access):** 1110 → 1109 → 1100 → 1101 → 1107 → 1102 → 1108 → 1105 → 1106 → 1103 → 1104 → 1111 → 1112 → 1113 → 1114.

---

## LA. Project kind, the shell, and the orange mode

**1000. [P0]** As a user creating a project, I want to choose what kind of project it is, so that the product configures itself for testing or for localization.
`UI:` `ProjectDialog` step 1 shows two equal cards — *QA & testing* and *Localization & content* — with one-line descriptions; the selected card gets the mode accent; step 2 shows kind-specific fields. Existing name / URL / environment fields remain.
`Agent/model:` none; `POST /api/projects` carries `kind`.
`Pass:` A project created from the Localization card has `kind: 'localization'` and opens in the localization shell; one created from the QA card is indistinguishable from today's projects; the dialog can be completed with keyboard only.
`Edge:` Kind is immutable after creation — the edit dialog shows it as a read-only badge with the hint "capabilities are additive; open the other mode's tabs from More".

**1001. [P0]** As a localization manager, I want to set source and target locales and the content I expect at creation, so that the agent, the QA rules and the workbench are correct from the first message.
`UI:` Searchable BCP-47 locale picker (typing "hin" finds `hi-IN`, flags shown as text codes, not emoji); target locales as removable chips; RTL locales get an "RTL" tag automatically; content-type checkboxes (website, mobile app, desktop app, e-learning/SCORM, documents, media, design files); "Add later" is allowed.
`Agent/model:` `Project.localization` written; per-locale defaults (plural rules, number/date patterns, expansion budget) filled from CLDR data.
`Pass:` Locales validate as BCP-47; duplicates and source-in-targets are rejected inline; `ar-AE` shows RTL and six plural categories in the per-locale drawer; the chat's first suggestion references the chosen content types.
`Edge:` Regional variants (`pt-BR` vs `pt-PT`, `zh-Hans` vs `zh-Hant`) are distinct locales with distinct TMs; script subtags are honored (`sr-Latn`).

**1002. [P0]** As a user, I want the shell to re-lay itself for localization — chat on the left, workbench on the right — so that the product reads as a translation workbench and not a QA console.
`UI:` In a `localization` project the left nav collapses to a 56px rail (project switcher, chats, search, settings), `MidPane` chat is the left column, `RightPane` mounts `LocWorkbench` with the tab strip **Assets · Workbench · Review · Player · Glossary & TM · Deliveries · Dashboard** and the live tabs **Browser · Mobile · Desktop** under More; right pane default width ~58% (`ts_pane_layout_loc`); `PaneHandle` still resizes; ⌘⇧F focuses the workbench full-window; empty states teach ("Drop a file, paste a URL, or connect Figma / GitHub").
`Agent/model:` none.
`Pass:` Switching from a QA project to a localization project in the switcher re-lays without reload; widths persist separately per mode; below 960px the workbench and chat share the width and the rail becomes a peek overlay; the QA layout is unchanged for `qa` projects.
`Edge:` A deep link to a localization tab (`?tab=loc-workbench`) on a `qa` project opens the tab in the QA shell (capabilities are additive) rather than erroring.

**1003. [P0]** As a user, I want the localization mode to look distinctly orange, so that I always know which half of the product I am in — and still have the same theme, contrast and components.
`UI:` `.app[data-mode="localization"]` overrides tokens only: dark `--accent: #F97316`, `--accent-hover: #FB923C`, `--on-accent: #0A0A0A`; light `--accent: #EA580C`, `--accent-hover: #C2410C`, `--on-accent: #FFFFFF`, accent text on white uses `#C2410C`; `--chart-1` orange; rail active indicator, focus ring, primary buttons, `ApprovalCard` eyebrow, progress bars and the brand lockup ("Studios · Localization") follow; 200ms crossfade on switch; `--danger`, `--pass`, `--warn`, `--live` unchanged.
`Agent/model:` none.
`Pass:` Every accent surface meets WCAG AA in both themes (measured in the e2e visual spec); QA projects render exactly as before (pixel diff of the QA shell = 0); Settings → Appearance offers "Reduce mode colours" which keeps the orange only on the brand lockup and rail indicator.
`Edge:` Screenshots and PDF reports generated in localization mode use the orange chart palette but keep severity colours identical to QA reports so a mixed org dashboard is readable.

**1004. [P1]** As a user of both modes, I want the kind visible wherever projects are listed, so that I never translate in a QA project or test in the wrong place.
`UI:` Kind badge (orange "L10N" / blue "QA") in the project switcher, recents, the mobile app project list, invite emails, and the enterprise portfolio pages; the org home shows two entry points ("Test something" / "Localize something").
`Agent/model:` `Project.kind` in list payloads; mobile create form gets the same two cards.
`Pass:` Badges appear on web, desktop and mobile lists; filtering the portfolio by kind works; the mobile app can create a localization project (name, kind, locales) even though it has no workbench.
`Edge:` Archived projects keep the badge but are excluded from mode-specific dashboards.

**1005. [P1]** As a QA lead with a localization project, I want the Browser, Mobile and Desktop live tabs available inside localization mode, so that validation happens where the live frames already are.
`UI:` Live tabs appear under More in the workbench strip; opening one shows the same `BrowserView` / `MobileView` / `DesktopLiveView` with the orange chrome; a "Validating hi-IN" chip sits in the tab header while a `loc_validate` job drives the surface.
`Agent/model:` `loc-validator` sub-agent uses `browser`, `mobile`, `desktop` groups.
`Pass:` A live frame streams in localization mode exactly as in QA; take-control works; the surface tab auto-opens when the agent starts a capture (`tab` event) unless the user pinned another tab.
`Edge:` If the project has no mobile builds, the Mobile tab shows the upload/Appetize onboarding rather than an empty frame.

---

## LB. Asset intake and management

**1006. [P0]** As a user, I want to drop any file or paste any URL into chat and have it become an asset, so that intake needs no form.
`UI:` Composer accepts drag-drop (files.ts `ACCEPT_FILES` extended with `.xlf .xliff .tmx .tbx .po .pot .resx .arb .strings .stringsdict .xcstrings .srt .vtt .idml .psd .fig .zip .mp4 .mp3 .wav`); each file shows a chip with detected format and size; unknown formats get an amber "Unknown format — treat as: [HTML] [Plain text] [Asset only]" choice.
`Agent/model:` `loc_ingest_asset` → `loc_ingest` job; format detection is deterministic (magic bytes, manifest names, JSON shape); no LLM for detection.
`Pass:` Dropping a SCORM zip, an `en.json` and a `.docx` in one message creates three assets; the progress card lists each with segment/word counts; the Assets tab updates live (`loc:asset` event); a wrong guess can be corrected from the asset's "Re-detect as…" menu without re-uploading.
`Edge:` A `.zip` that is not SCORM (a folder of PNGs, a Lokalise export of JSON files) is ingested as a bundle: each inner file becomes a document with its own filter.

**1007. [P0]** As a localization manager, I want an Assets tab that shows what I have, where it is and how far each locale is, so that I can run a program from one list.
`UI:` `DataTable` columns: name, kind icon, version, documents, words, per-locale progress mini-bars (translated / reviewed / approved), flags (art needed, source pending), last activity; `FilterBar` (kind, locale, status, flag); `ViewSwitch` table/cards/tree; `StatStrip` totals; row click opens a `DetailPage` drawer with documents, history, deliveries.
`Agent/model:` reads `loc_assets` / `loc_documents`.
`Pass:` 500 assets render without jank (virtualized); filters persist per user; the per-locale bars match the Workbench counts exactly; "Open in Workbench" from a row lands on that asset's first untranslated segment for the selected locale.
`Edge:` A stale asset (a newer version exists) shows a "v2 available" chip and a compare action instead of silently hiding v1.

**1008. [P0]** As an e-learning localizer, I want SCORM packages (1.1 CSF, 1.2, 2004) ingested with their structure, so that every translatable string has a home and nothing outside the package is invented.
`UI:` Asset drawer shows the manifest tree (organizations → items → SCOs → resources), per-SCO segment counts, "18 images contain text", "12 audio files", the SCORM version and manifest validity; Player tab is enabled once ingest completes.
`Agent/model:` `packages/loc-formats` SCORM filter: manifest parser, DOM-aware HTML extraction (`title`, text nodes, `alt`, `aria-label`, `placeholder`, `value` on inputs), `questions.js` / `contentfunctions.js` string extraction with quiz answer linkage, `xml:base` resolution.
`Pass:` The Rustici "Golf Explained" sample yields the manifest titles, all page text, quiz stems/choices with `linkedKey`, and flags its images; segment keys are stable across two ingests of the same zip; a malformed manifest is reported with line/column and the package still ingests as an HTML bundle.
`Edge:` Common-root folders, `__MACOSX`, `.DS_Store`, mixed-case `imsmanifest.xml`, and HTML with inline `<script>` string tables are handled; SCORM 2004 sequencing rules are preserved byte-for-byte on export.

**1009. [P0]** As a developer, I want strings files and XLIFF ingested with their semantics, so that plurals, arrays, comments and existing translations survive a round trip.
`UI:` Documents show format badges (JSON i18next, ARB, Android XML, iOS `.strings`/`.stringsdict`/`.xcstrings`, PO/POT, RESX, YAML, `.properties`, XLIFF 1.2/2.x); plural sets render as one grouped segment with one row per CLDR category; developer comments/notes show as segment context; ICU arguments are inline-code chips.
`Agent/model:` deterministic filters; XLIFF states map to segment status (`new`→empty, `translated`→draft, `reviewed`→reviewed, `final`→accepted).
`Pass:` Export of an unmodified asset is byte-identical modulo the translated values (key order, comments, indentation, quoting, BOM and line endings preserved); an XLIFF 2.1 file with `<mrk>` and `<ph>` round-trips through `typesxliff` without loss; Android `string-array` order is preserved.
`Edge:` Duplicate keys across files, keys with dots vs nested objects, escaped quotes in `.strings`, `%1$s` vs `{0}` positional placeholders, and `xcstrings` with per-device variations.

**1010. [P1]** As a content owner, I want Office documents ingested with formatting kept, so that a translated DOCX or PPTX looks like the original.
`UI:` Document preview pane shows the rendered page/slide with translated runs highlighted; tables, notes and alt text appear as tagged segments; PDF assets show a "no writeback" notice with the available outputs (DOCX / HTML / side-by-side).
`Agent/model:` OOXML filters (`word/document.xml` runs merged per paragraph, `ppt/slides/*.xml`, `xl/sharedStrings.xml`); PDF text via `pdf-parse` for review only.
`Pass:` Bold/italic/link runs in a paragraph survive translation as inline codes and re-apply to the translated text; slide notes and speaker notes are separate documents; a 200-page DOCX ingests under 30s; PDF produces a translated DOCX and a side-by-side HTML review.
`Edge:` Tracked changes and comments in DOCX are ingested from the accepted text only and flagged; merged cells and formulas in XLSX are left untouched (only string cells are segments).

**1011. [P0]** As a marketer, I want a website ingested from its URL, so that pages become documents without exporting anything from the CMS.
`UI:` "Paste a URL" in chat or Assets → Add → URL; a wizard card shows discovered locales (from `hreflang`, sitemap alternates, nav, probing), the URL strategy (path / subdomain / query / cookie), page count and a bounded crawl depth; pages appear as documents with screenshots as context.
`Agent/model:` `loc_site_discover_locales` (ported GTW discovery), `BrowserSession` crawl bounded by `Project.exploration`, HTML filter with visible-text extraction and `data-*`/`aria` attributes; access-model lanes for auth walls (never evades protection).
`Pass:` For a site with `/en` and `/fr` the wizard lists both with the strategy "path-prefix"; only in-scope hosts are crawled; a login wall produces a HITL "Sign in for me / Use saved session" card instead of a failure; dynamic text rendered by JS is captured after network idle.
`Edge:` Infinite-scroll and cookie-consent overlays (dismissed via known selectors, else flagged), cross-domain redirects (only if allowed), duplicate pages by canonical URL, and pages that switch locale via cookie only.

**1012. [P1]** As a designer or engineer, I want assets pulled from connectors — Figma frames and text nodes, a GitHub path, CMS entries — so that intake follows the source of truth.
`UI:` Assets → Add → Connector shows the presets connected for the project (Figma file/page picker with frame thumbnails when the plan allows, GitHub repo/branch/path picker, Contentful/Webflow/WordPress content-type picker); imported items keep a source link and a "Re-sync" action.
`Agent/model:` MCP tools from the connector presets; Figma text nodes read via REST (`characters`, style, bounding box) with frame renders cached (free plans allow very few image requests per month).
`Pass:` A Figma frame yields one document with text-node segments and a context screenshot; a GitHub path yields strings-file documents pinned to a commit SHA; "Re-sync" ingests a new asset version rather than mutating the old one.
`Edge:` Figma rate limits surface as a precise message with the retry time; renamed Figma nodes are matched by node id, not text; a GitHub path that no longer exists marks the asset "source moved".

**1013. [P0]** As a localization manager, I want a changed source to become a new version with a clear diff and kept leverage, so that a v2 course or a new build does not restart the work.
`UI:` Asset drawer → Versions: v1, v2 with a diff summary card (new / changed / removed / moved segments, words affected per locale); "Carry over" preview lists what will be reused; both versions open side by side in the Workbench.
`Agent/model:` `loc_ingest` with re-alignment by key, then by content hash for moved files, then fuzzy (trigram ≥ 0.85) with the previous target offered as a TM match; no LLM in alignment.
`Pass:` Unchanged keys keep translations and statuses; changed segments become `draft` showing the old target as a TM hit with its score; removed segments are archived (not deleted) and excluded from exports; the diff counts equal what the Workbench filters show.
`Edge:` Renamed documents matched by hash; key collisions resolved by locator order and reported; a v2 that removes a whole locale from the manifest asks before dropping approved deliveries.

**1014. [P0]** As an administrator, I want intake to be safe and predictable for big or hostile files, so that a 400MB bundle or a crafted zip never takes the service down.
`UI:` Upload shows chunked progress with pause/resume; caps are stated in the drop zone (documents 25MB, bundles/media 400MB); rejected files show the precise reason ("zip entry escapes the package root", "compression ratio 1:2000").
`Agent/model:` streaming zip reader, entry-count and inflated-size limits, path-traversal rejection, encoding detection (UTF-8/16, BOM), size caps from `artifacts.ts`, ingest runs in the worker not the API request.
`Pass:` A 380MB SCORM package with video ingests without the API process exceeding its memory budget; a zip with `../` entries is rejected before extraction; a 1:2000 compression bomb is rejected; Latin-1 HTML is transcoded and flagged.
`Edge:` Resumed uploads after a network drop; two identical uploads deduplicate by content hash with a "already ingested as v3" notice.

**1015. [P0]** As a linguist, I want segments to be sentence-sized with placeholders and tags protected, and to carry context, so that I translate meaning, not markup.
`UI:` Inline codes render as compact chips (`{count}`, `<b>…</b>`, `%1$s`) that cannot be edited character-wise, only inserted/moved; the drawer shows the context screenshot with the segment's region highlighted, the developer note, and `maxLength` when known; the grid shows a small "3 codes" indicator.
`Agent/model:` SRX-lite segmentation rules per language (abbreviations, CJK punctuation, no split inside codes); context screenshots come from `loc_capture_screen` or the source build run (`loc-validator` maps keys to on-screen text), else from the document preview.
`Pass:` A paragraph with two sentences becomes two segments that export back into one paragraph; whitespace at segment edges is preserved; every inline code in the source is present in a confirmed target or the segment carries a QA flag; the context screenshot opens at the right region.
`Edge:` Segments that must not be split (addresses, code samples, `pre` blocks) are marked by the filter; CJK sources without spaces; strings with embedded newlines and tabs.

---

## LC. Terminology, translation memory, and style

**1016. [P0]** As a terminologist, I want to import a termbase (TBX, CSV, XLSX) and edit terms, so that the agent and the QA checks use the client's words.
`UI:` Glossary & TM tab → Termbases: import dialog with column mapping preview for CSV/XLSX; term table with source, per-locale targets, DNT toggle, forbidden terms, category (brand/legal/ui/general), definition, notes, authority (client / regulator / internal); inline edit; `FilterBar` by locale/category/DNT.
`Agent/model:` `loc_termbase_import` (ported TBX parser, DOM-aware), `loc_term_add`; no LLM.
`Pass:` A TBX with three languages imports all pairs and reports skipped entries with reasons; DNT terms ("PAN", brand names) appear as protected chips in the Workbench when present in a source; a forbidden term in a target raises a `terminology` QA flag within one second of typing.
`Edge:` Case-sensitivity per term, multi-word terms, inflected forms in morphologically rich languages (stem matching with a "fuzzy term" indicator), and conflicting entries across two bound termbases (client wins, with a conflict badge).

**1017. [P1]** As a localization manager, I want the agent to propose a glossary from the content itself, so that a new client starts with terminology instead of discovering it in review.
`UI:` Chat: "Generate a glossary for this asset" → a review card listing proposed terms grouped by category with per-locale suggestions, DNT candidates and preserve rules; each row can be accepted, edited or dropped; nothing is active until "Activate glossary".
`Agent/model:` `loc_generate_glossary` on the `glossary.generate` task type (Balanced tier), grounded on the asset text and any existing termbase; proposals carry a confidence and an example sentence.
`Pass:` For a 10k-word course the card proposes 40–80 terms in under two minutes; accepted rows land in a new termbase bound to the project; rejected rows are remembered so they are not proposed again for this project.
`Edge:` Regulated domains (finance, medical) mark terms "needs authority" when no external source is cited; the generator never invents translations for DNT terms.

**1018. [P0]** As a localization manager, I want to import and scope translation memories, so that prior work is reused and never leaks across clients.
`UI:` Glossary & TM → Memories: import TMX / XLIFF (bilingual), choose scope (this project / org-wide), see unit counts per locale pair, last used, and a leverage report against the current assets ("42% exact, 18% fuzzy ≥ 85%").
`Agent/model:` `loc_tm_import`; units hashed and embedded (`embed.tm`) in the worker; scope is enforced in every query.
`Pass:` Importing a 100k-unit TMX completes as a resumable job with progress; the leverage report matches the pre-translate estimate; a project-scoped TM is invisible to other projects (verified by a cross-project search returning nothing).
`Edge:` TMX with mixed locale codes (`en-US` vs `en`) normalizes with a warning; duplicates within an import are collapsed; re-embedding when the embedding model changes runs as a background job with the old vectors kept until the new ones finish.

**1019. [P0]** As a linguist, I want TM matches shown with their score and origin at the moment I need them, so that I reuse instead of retyping.
`UI:` Drawer → Matches: exact (100%), in-context exact (101%, same neighbours), fuzzy (75–99%) with diff highlighting against the current source, semantic matches labelled "meaning" with a distinct icon; origin (human / MT / imported), date and project; ⌘1–9 inserts; a "Pre-fill all ≥ 95% as draft" bulk action per asset.
`Agent/model:` `loc_tm_search` (hash → trigram → embedding); optional auto-confirm of 101% matches as `reviewed` if the project policy allows.
`Pass:` Matches appear in under 300ms for a 100k-unit TM; the diff highlight shows exactly what differs; inserting a fuzzy match sets origin `tm` and match score on the segment; auto-confirmed 101% segments are visibly marked so a reviewer can undo them in one filter.
`Edge:` Matches whose inline codes differ from the current source are shown with the codes re-mapped and a "codes adjusted" note; low-quality units (rejected in review) are demoted, never deleted.

**1020. [P0]** As a reviewer, I want terminology and TM to shape the machine translation, so that the agent's first draft already speaks the client's language.
`UI:` Segment drawer shows "Used: 3 terms, 1 TM hit" on MT drafts; hovering shows which; the Estimate card states how many segments will be MT vs TM vs post-edit.
`Agent/model:` `loc_translate` prompt assembly injects bound terms found in the batch, 100% / high-fuzzy matches as constraints, DNT lists, style settings and a few confirmed neighbours as examples; `postedit.fuzzy` task for 85–94% matches.
`Pass:` A draft that contradicts a bound term is not produced without a `terminology` flag; a DNT term appears verbatim; the recorded prompt version on the segment shows the terms that were injected.
`Edge:` Batches with more terms than fit the context window are split by term overlap; conflicting term and TM guidance prefers the termbase and notes it.

**1021. [P1]** As a brand owner, I want a per-locale style guide (formality, voice, do/don't) that the agent applies and remembers, so that quality is consistent across assets.
`UI:` Project Settings → Localization → Style per locale: formality (formal / neutral / casual), voice notes, forbidden constructions, examples; the chat can propose additions ("You corrected 'Sie' to 'du' 12 times — make casual the default for de-DE?") as a memory proposal card.
`Agent/model:` settings injected into every translate/review prompt; accepted proposals become team-layer memory facts tagged `loc`; content itself is never stored in memory (`guardMemory`).
`Pass:` Switching de-DE from formal to casual changes the next batch's output on a sample of 20 segments; the memory fact appears in Memory view under the team layer with the `loc` tag; deleting it stops the behaviour.
`Edge:` Style conflicts between org and project settings resolve project-first and show both; locale-specific typographic rules (French spaces before `:`, German quotes) are part of style and enforced by QA (1053).

**1022. [P1]** As a linguist, I want concordance search across TMs and a tidy TM, so that reuse is fast and old junk does not pollute suggestions.
`UI:` ⌘⇧K opens concordance: search source or target across bound TMs with context; results insertable; Glossary & TM → Memories has "Hygiene": duplicates, units with rejected reviews, units older than N years, one-click demote/archive with counts.
`Agent/model:` `loc_tm_search` in concordance mode; hygiene is deterministic.
`Pass:` Searching "purchase order" returns bilingual hits with the containing sentence and asset; hygiene demotes flagged units so they stop appearing above fresh matches; nothing is hard-deleted without confirmation and an audit entry.
`Edge:` Very short segments ("OK", "Yes") are excluded from semantic matching to avoid noise; cross-locale contamination (a `pt-PT` unit offered for `pt-BR`) is prevented unless the project opts into a fallback with a visible "from pt-PT" label.

---

## LD. Model routing via OpenRouter

**1023. [P0]** As an admin, I want a routing table that maps each localization task type to a model tier, so that translation, review, vision, image, video and speech each use the right model without anyone hand-picking per job.
`UI:` Settings → Models → Localization: rows per task type (see reference table) with tier (economy / balanced / best), the resolved model, its price per unit, latency class and a "Recommended" badge; per-locale overrides in a collapsible sub-table; "Reset to recommended".
`Agent/model:` `Project.models.loc[taskType]` and `Project.localization.mtTiers[locale]`; the router applies org allowlist → data-policy filter → tier → per-pair prior → cost tie-break.
`Pass:` Changing `review.mqm` to a specific model is reflected on the next QA job's records; the resolved model for `translate.ui` on `hi-IN` differs from `en→fr` when priors differ; the review model is never the same as the translation model for the same batch (the UI blocks it and explains why).
`Edge:` A model removed from OpenRouter mid-project resolves to the next best in the same tier and posts an `llm` notice; a locale with no prior falls back to the tier default.

**1024. [P0]** As a localization manager, I want the product to suggest the best current model for each language pair with the reason, so that I trust the default and can defend the cost.
`UI:` In the routing table and in the Estimate card, each suggested model shows a reason line: "Top in your org's de-DE evals (MQM 8.7 on 240 segments), ZDR provider, $0.9 / 1k words"; alternatives shown on hover with the delta.
`Agent/model:` recommendation = weighted score of (org A/B results per pair, 1027) + (catalog metadata: context, modalities, structured output support, price) + (policy fit); recomputed nightly and on catalog change; explanation is templated, not generated.
`Pass:` The recommended model for a pair changes after an A/B that beats it by the configured margin; the reason line cites the eval it came from; a pair with no evals says so and uses the tier default.
`Edge:` Ties resolve to the cheaper model; a model that failed structured output more than 2% of the time in the last 7 days is demoted with the failure rate shown.

**1025. [P0]** As an admin, I want model discovery filtered by capability and policy, so that only models that can do the job and meet our data rules are ever selectable.
`UI:` Model picker shows capability chips (text, image-in, image-out, video-out, audio-in, audio-out, structured output, 128k+); a "ZDR only" toggle; org allowlist status; price and context; models outside policy are shown greyed with the blocking reason.
`Agent/model:` catalog from `/api/v1/models` + `/images/models` + `/videos/models` cached in `model-catalog.ts` with `output_modalities`, `supported_parameters` and provider data-policy fields.
`Pass:` Selecting `image.edit` lists only models with image output and `input_references`; with "ZDR only" on, the translate tier resolves only to ZDR providers and the job record shows the provider; a blocked model cannot be set via API either.
`Edge:` Catalog fetch failures keep the last good catalog with a "stale since" indicator; per-user BYOK keys inherit the org policy.

**1026. [P0]** As a localization manager, I want low-confidence or QA-failed segments escalated to a stronger model and provider outages handled, so that quality does not depend on luck.
`UI:` Progress card shows "escalated 14 segments to best tier" with the cost delta; segment history shows both attempts; a job paused by an outage shows the reason and "Resume with fallback".
`Agent/model:` translator emits a per-segment confidence; segments failing deterministic QA or below the confidence threshold are retried on `escalateTo`; provider errors trigger the same-tier fallback list; `budgetGate` caps escalation spend.
`Pass:` Escalated segments pass QA at a higher rate than the first attempt in the job summary; no segment is billed twice for the same attempt; an induced 503 from the primary provider does not fail the job.
`Edge:` If the fallback is the same model on another provider, the record shows the provider change; escalation is skipped when it would exceed the envelope and the segment is flagged for a human instead.

**1027. [P1]** As a localization manager, I want to A/B models on a real sample before committing a locale, so that the model choice is evidence, not opinion.
`UI:` Chat: "Compare models for ja-JP on this asset" → picker of 2–3 candidates → results card: per model MQM-lite score, back-translation spot checks, cost, latency, and 5 example segments side by side; "Set as default for ja-JP" button.
`Agent/model:` `loc_translate` with `sample=50`, `review.mqm` by a third model, results stored as an org eval that feeds 1024 priors.
`Pass:` The card's winner becomes the recommended model for the pair after confirmation; the sample segments are marked `draft` and not billed to the campaign; the eval is visible in Settings → Models → Evals with its date.
`Edge:` Samples are stratified across documents and lengths; the same sample is reused when re-running so comparisons stay fair.

**1028. [P0]** As a localization manager, I want to see the cost of each task type in units I understand before it runs, so that estimates are meaningful for content, not tokens.
`UI:` Estimate card lines: words × locale × model price, TM leverage discount, images (count × price), audio/video seconds; per-line model shown; a "Cheaper / Better" toggle recomputes with the other tier; totals against the envelope with the checkpoint marks.
`Agent/model:` `loc_estimate` prices from `pricing.ts` (tokens/word factor per language) and media prices from the catalog; `UsageRow.units` recorded after the run for reconciliation.
`Pass:` The estimate is within ±15% of the recorded spend for text on a completed job (shown as "estimate accuracy" on the Dashboard); switching tiers changes both the model names and the totals on the card.
`Edge:` Languages with high token-per-word ratios (Thai, Japanese, Hindi) use per-language factors, not a global average; media whose duration is unknown until transcription show a range.

**1029. [P1]** As an engineer, I want repository work done by my own coding CLI agent and my own model keys where I have them, so that engineering tasks cost nothing extra and stay in my environment.
`UI:` Model picker offers "Claude Code / Cursor / Codex / OpenCode (installed)" for `engineering` tasks and BYOK status for OpenRouter (Settings → LLM); the `loc-engineer` card in chat says which provider ran and links the PR.
`Agent/model:` `packages/cli-agents` drivers run in the project workspace with the repo checked out; OpenRouter key resolution via `llm-keys.ts` (per-user key → shared key).
`Pass:` With Claude Code installed, "externalize the hardcoded strings" runs there at $0 recorded cost; without any CLI agent the task falls back to an OpenRouter model with a notice; a revoked BYOK key surfaces the existing `llm` banner rather than a silent failure.
`Edge:` CLI agent output is treated like any tool output (redacted, size-capped); the PR is never opened without the HITL approval card.

---

## LE. Agent translation workflow

**1030. [P0]** As a localization manager, I want an estimate and a campaign proposal before any spend, so that I approve cost and order of work, not a surprise.
`UI:` After ingest the agent posts an Estimate card (1028) then a `CampaignCard` (variant `localization`): items = assets × locales, tranches (e.g. A: pilot locale, B: the rest), cap, checkpoints; Edit / Approve tranche A; the campaign appears in Governance and in the Jobs strip.
`Agent/model:` `propose_campaign` with item kind `localization`; `approveCampaign` enqueues `loc_translate` and `loc_qa` jobs with campaign refs; `envelopes.ts` enforces caps.
`Pass:` Nothing translates before approval (a sample preview of ≤ 10 segments is allowed and labelled); tranche B does not start until A's delivery is approved when so configured; the budget checkpoint card appears at 50/80/100%.
`Edge:` Editing locales or assets in the card re-prices before approval; a cap below the estimate is allowed with a warning and results in a partial, resumable job.

**1031. [P0]** As anyone, I want translation to run as a job with live progress I can pause, resume or cancel from any device, so that long work is never trapped in a chat tab.
`UI:` `LocProgressCard` in chat: per-locale bars (segments done / total, words, escalations, flags), ETA, cost so far, Pause / Resume / Cancel; the same job in the Jobs strip and on mobile; the Assets row bars animate.
`Agent/model:` `loc_translate` job; checkpoints per batch (`caseIndex` = batch, `state` = last segment ids); idempotency key (asset, locale, segment hash, prompt version).
`Pass:` Killing the worker mid-job and restarting resumes from the last batch with no duplicate spend (usage rows are unique per idempotency key); pausing from the phone stops the next batch within one batch; cancel leaves completed segments as `draft`.
`Edge:` Partial translations (only empty / rejected, a selection, one file) are the same job with a filter; re-running on the same filter is a no-op with a "nothing to do" summary.

**1032. [P0]** As a linguist, I want the agent to translate with context and strict output, so that drafts are coherent and never corrupt the file.
`UI:` Segment drawer → "How this was translated": batch neighbours used, document title, screenshot presence, model, prompt version; malformed-output retries are visible in job logs, not in the grid.
`Agent/model:` batches packed by document with neighbour segments, title and context notes; strict JSON schema (`{i, text}` per item); validation of counts and ids; automatic re-ask on malformed output; temperature low for `translate.ui`.
`Pass:` Every batch returns exactly one target per input or is retried; no target contains a stray JSON fragment or another segment's text; a 5k-segment asset completes with fewer than 1% batch retries in the job summary.
`Edge:` Very long segments (> 2k words) are translated alone; batches never split a plural set or a quiz group; models without JSON mode use a tool-call schema instead.

**1033. [P0]** As a developer, I want placeholders, tags and ICU arguments preserved in every draft, so that a translated app never crashes or shows `{count}` literally.
`UI:` Inline code chips in target; a missing or extra code shows an inline red chip "missing {count}" with one-click insert-at-caret; the `placeholder` QA flag appears in the grid; export is blocked for `critical` placeholder flags by default policy.
`Agent/model:` codes are masked before MT and restored after; deterministic parity check on every draft; auto-repair when the mapping is unambiguous, otherwise flag.
`Pass:` Across the fixtures no exported file has a placeholder mismatch; positional placeholders reordered legitimately for the target grammar (`%2$s %1$s`) pass; HTML tags stay balanced.
`Edge:` Placeholders that look like words (`{name}` vs `{Name}`), nested tags, escaped braces (`'{'` in ICU), and RTL segments where codes visually reorder but logically stay.

**1034. [P0]** As a developer, I want plural forms generated and validated per target language, so that Arabic gets six forms and Japanese gets one without anyone remembering the rules.
`UI:` Plural segments render as a group with one row per CLDR category required by the locale (zero / one / two / few / many / other) and example numbers per row; missing categories are pre-created empty with an amber flag; extra categories the source has but the target does not need are greyed.
`Agent/model:` CLDR plural rules per locale in `loc-formats`; the translator prompt lists required categories with example counts; the ICU/`stringsdict`/Android writer emits exactly the target's categories.
`Pass:` `ar-AE` exports `zero, one, two, few, many, other`; `ja-JP` exports only `other`; `ru-RU` exports `one, few, many, other`; a plural with a missing required category cannot be `accepted`.
`Edge:` Ordinal vs cardinal plurals; `select` (gender) messages nested in plurals; sources that abuse plurals for non-count strings.

**1035. [P0]** As a designer, I want length budgets respected and overflow found before translation, so that German does not overflow a button the agent has seen.
`UI:` Segment shows `maxLength` (from context capture or file metadata) and a live length meter (chars and estimated pixel width using the captured font size); over-budget drafts get a `length` flag and the agent's shorter alternative in the drawer ("Bestätigen" for "Überweisung bestätigen"). Assets → "Run pseudo-locale" builds a `qps-ploc` variant (accented characters, +35% expansion, `[brackets]` marking string edges) and captures it on the Browser/Mobile tabs; overflow and concatenation findings appear in Review before any locale is translated.
`Agent/model:` translator receives `maxLength` and the UI element type; on overflow it produces a shorter variant; the `layout-fit` check uses expansion tables per language when no budget exists; `loc_pseudo_localize` is deterministic and its capture run is an ordinary `loc_validate` job.
`Pass:` For every segment with a `maxLength`, the accepted target fits or carries an explicit reviewer override note; the pixel estimate is within 10% of the measured width in the captured screenshot for Latin scripts; the pseudo-locale run on the fixture finds the seeded truncation and a hardcoded (unbracketed) string.
`Edge:` Scripts with different metrics (CJK, Devanagari) use per-script width factors; abbreviations are only proposed from the termbase or with a `needs-review` flag; RTL strings measured logically; pseudo-locale builds are never exported as deliveries.

**1036. [P1]** As a marketer, I want transcreation variants for creative copy with back-translations, so that I choose a message rather than accept a translation.
`UI:` Segments tagged `transcreate` show three variants (A/B/C) with literal back-translations and a one-line rationale each; pick one, edit, or ask for three more; picked variant sets origin `human-selected`.
`Agent/model:` `transcreate` task type (best tier, higher temperature, 3 variants), `backtranslate` by a different model.
`Pass:` Variants differ meaningfully (not paraphrases of one idea, checked by embedding distance threshold); back-translations are literal (reviewer-verified in evals); the chosen variant propagates to identical taglines elsewhere in the asset only after confirmation.
`Edge:` Legal disclaimers tagged `legal` are excluded from transcreation even if they sit in a marketing page; brand names in taglines remain DNT.

**1037. [P1]** As an e-learning localizer, I want linked and identical segments kept consistent, so that a quiz still grades and the same button reads the same everywhere.
`UI:` Linked segments (quiz choice ↔ correct answer) show a link icon; editing one shows "also updates 1 linked segment"; identical sources across the asset show a "×7" repetition badge with "Propagate" (confirm) and "Keep unique here" options; the consistency audit (1052) lists divergences.
`Agent/model:` `linkedKey` from the SCORM filter; repetition detection by normalized source; propagation is deterministic.
`Pass:` Changing a quiz choice updates the answer key so `scorm_walk` still scores the quiz; propagating a target to seven repetitions updates all seven with origin `propagated`; a segment marked "keep unique" is excluded from later propagations.
`Edge:` Same source with different context notes is not treated as a repetition; propagation across locales is never offered.

**1038. [P1]** As a localization manager, I want the agent to learn our decisions and never our content, so that quality compounds and confidentiality holds.
`UI:` Memory proposals appear as cards ("Remember: es-MX uses informal tone for CTAs?"); Memory view shows `loc` facts under team and user layers with source links; a "Content never stored" note in Settings → Memory → Localization.
`Agent/model:` curator extracts style/rule facts only; `guardMemory` rejects facts containing segment text longer than a short phrase or any PII; TM is the store for content.
`Pass:` After 10 corrections toward casual tone the agent proposes a fact; accepting changes subsequent drafts; grepping memory for any 12-word sequence from the assets returns nothing.
`Edge:` Facts are per locale; contradictory facts prompt a resolution card rather than silently overwriting.

**1039. [P1]** As a linguist, I want the agent to ask when a segment is ambiguous rather than guess, so that gendered, polysemous or context-free strings are resolved once.
`UI:` Sticky HITL card above the composer: the segment, the screenshot, the question ("Is 'Save' a verb or a noun here?"), two to four options and a free-text field; answering resolves all segments the answer applies to; unanswered questions stay in the Review queue.
`Agent/model:` `request_user_help` kind `question` with `segmentIds`; batching of similar questions; the answer is recorded as segment context and, if general, as a memory fact.
`Pass:` The card lists how many segments the answer affects; answering updates their drafts in the same job; the question does not recur for the same key.
`Edge:` Questions are suppressed when the context screenshot already resolves them (the agent must cite why it still asks); more than five open questions collapse into one card with a list.

---

## LF. The Workbench (CAT editor)

**1040. [P0]** As a linguist, I want a segment grid that stays fast on large assets, so that a 10k-segment course feels like a spreadsheet, not a web page.
`UI:` Virtualized grid (`@tanstack/react-virtual`) with sticky header; columns: #, source, target (active locale), status chip, flags, TM score, assignee; row height adapts to content; column widths persist; second locale column optional for cross-locale review.
`Agent/model:` none.
`Pass:` Scrolling 10k rows holds 60fps on a mid laptop; jump-to-segment by number or key is instant; selection survives filtering; the grid is keyboard navigable and screen-reader labelled.
`Edge:` Extremely long segments expand inline with a "show more" instead of blowing row height; RTL targets align right per row without affecting the source column.

**1041. [P0]** As a linguist, I want an editor that treats codes as objects and handles RTL and whitespace properly, so that I cannot break markup by accident.
`UI:` Target editor with inline chips for codes (insert with ⌘⇧1–9, drag to move, delete as a unit), whitespace visualizer toggle (·, ⇥, ¶), proper bidi editing with `dir` per segment and a logical/visual caret indicator, non-breaking space insertion (⌥Space), typographic quote auto-pairs per language.
`Agent/model:` none.
`Pass:` It is impossible to produce a target with half a tag; typing Arabic in a target keeps codes in the right logical position; the visualizer shows trailing spaces the source has.
`Edge:` Paste from Word strips styling but keeps line breaks; emoji and combining marks count as one grapheme in length meters.

**1042. [P0]** As a linguist, I want a keyboard-first flow, so that I never leave the home row to translate.
`UI:` ⌘↵ confirm and go to next unconfirmed; ⌘↑/↓ move; ⌘1–9 insert TM match; ⌘I insert highlighted term; ⌘⇧1–9 insert code; ⌘K command palette (filters, jump, actions); ⌘B log defect from segment; ⌘E toggle preview split; ⌘⇧K concordance; ⌘/ shortcut sheet; all mirrored in a "Shortcuts" popover.
`Agent/model:` none.
`Pass:` A tester can confirm 100 pre-filled segments with only the keyboard; shortcuts do not collide with browser or Electron defaults on macOS/Windows; every shortcut has a visible menu equivalent.
`Edge:` Shortcuts pause inside the composer and dialogs; IME composition (Japanese/Chinese input) never triggers a shortcut mid-composition.

**1043. [P0]** As a linguist, I want a right drawer with everything about the current segment, so that I do not hunt across tabs.
`UI:` `Drawer` sections: Matches (1019), Terms (highlighted in source, click to insert), QA flags with fix actions, Context (screenshot with region, note, `maxLength`), Back-translation (on demand), History (who/what/model/when, revert), Comments; sections collapsible and order persisted.
`Agent/model:` `loc_tm_search`, `loc_check_glossary`, `loc_back_translate` (Balanced tier) on demand; history from segment `rev` log.
`Pass:` The drawer updates within 300ms of moving rows; clicking a term inserts at caret; revert restores text, status and origin from history; the drawer never blocks the grid on slow calls (sections load independently).
`Edge:` Segments with no context show a "Capture context" action that queues the segment for the next validation run.

**1044. [P1]** As a reviewer, I want filters that match how I work — mine, flagged, changed since, by tag — and saved views, so that I can review in passes.
`UI:` `FilterBar` chips: status, flag type, tag (`quiz`, `legal`, `ui`), locale, origin (MT / TM / human), assigned to me, changed since (date or "since my last visit"), search in source/target/key; saved views per user with names; result count and word count in the header.
`Agent/model:` none.
`Pass:` "Flagged + not mine" returns the right set (cross-checked against the QA job summary); saved views survive reload; "Accept all in view" respects the current filter exactly and asks for confirmation with counts.
`Edge:` Filters across two locales at once for parity review (e.g. `de` accepted but `fr` empty).

**1045. [P0]** As a reviewer, I want a live preview that shows the translation in place, so that I judge fit and meaning together.
`UI:` Bottom/split preview per document type: HTML/SCORM page with server-side overlay (no re-zip), image with rendered text, video with the subtitle track, DOCX/PPTX page render; clicking text in the preview jumps to its segment; segment hover highlights its region; locale switcher in the preview header.
`Agent/model:` overlay file server (`GET /api/loc/assets/:id/files/*?locale=`) cached by segment `rev`; live-site overlay uses the browser tab with an injected locator map.
`Pass:` Editing a segment updates the preview within a second; a segment whose locator no longer resolves (source changed) is marked "not in preview" instead of failing the page; the preview is sandboxed (no package script can reach the app).
`Edge:` Pages that depend on JS-generated text show the overlay where locators match and a note where they do not; video previews scrub to the segment's timestamp.

**1046. [P1]** As a team, we want to work in the same asset at once without stepping on each other, so that review scales.
`UI:` Presence avatars on documents and rows; a row being edited by someone else shows a soft lock with their name; conflicting saves show a diff card "Rahul saved a different version — keep mine / take theirs / merge"; @mentions in comments notify.
`Agent/model:` `rev` per segment with optimistic concurrency; `live` events `loc:segment`; agent writes obey the same locks (the agent never overwrites a human edit made after its read).
`Pass:` Two users editing different rows never block each other; editing the same row produces the conflict card, never silent loss; an agent job skips rows a human edited after the job started and lists them.
`Edge:` Offline for 30s then reconnect merges cleanly or shows conflicts; presence expires within 60s of a closed tab.

**1047. [P1]** As a reviewer, I want bulk actions that are precise and reversible, so that I can move a file forward in one gesture and undo one mistake.
`UI:` Selection checkboxes with shift-click ranges; actions: Accept, Reject with note, Propagate, Re-translate with model picker (per task type), Clear targets, Assign, Add tag; a confirmation lists counts by status; an undo toast for 10 seconds; the action is one history entry.
`Agent/model:` bulk endpoints; re-translate enqueues a filtered `loc_translate` job.
`Pass:` Accepting 300 selected segments creates one audit entry with 300 children; undo within 10s restores all statuses; re-translate on a selection bills only those segments.
`Edge:` Bulk actions on a filtered view that changed underneath (someone accepted rows meanwhile) re-validate and report the difference.

**1048. [P1]** As an auditor, I want segment history with diffs, so that I can see who or which model changed what, and revert.
`UI:` History section: entries with actor (user / agent + model + prompt version), timestamp, text diff, status change, origin; Revert; "Export history" for an asset.
`Agent/model:` every write records `rev` metadata including `generationId` for model writes.
`Pass:` A segment translated by MT, edited by a human, then propagated shows three entries with correct actors; reverting to the MT entry restores its origin and match score; the export lists model costs per entry.
`Edge:` Histories longer than 50 entries paginate; bulk actions show as one parent entry expandable to children.

---

## LG. Linguistic QA

**1049. [P0]** As a linguist, I want deterministic QA checks on every write and every batch, so that the mechanical errors never reach review.
`UI:` Flags as compact chips in the grid (`placeholder`, `tags`, `plural`, `number`, `date`, `currency`, `untranslated`, `empty`, `length`, `whitespace`, `punctuation`, `quotes`, `dnt`, `forbidden-term`, `double-space`); drawer shows the rule, the evidence and a one-click fix where safe; Settings → Localization → QA rules lets a project disable or tune a rule per locale with a reason.
`Agent/model:` `loc_qa_check` rules in `loc-formats` (no LLM): parity of codes, tag balance, CLDR plural completeness, number/date/currency patterns per locale, target == source when not DNT, length ratio bounds per language, whitespace/punctuation parity, locale quotes, DNT preserved, forbidden terms.
`Pass:` Typing a target with a missing `{count}` flags within one second; the batch QA summary counts match the grid filter; a disabled rule shows as "disabled by Meera: reason" on affected segments; the rule set has fixture tests per locale.
`Edge:` Legitimate differences (Hindi digits vs ASCII digits when the project allows both, French non-breaking spaces before `?`) are configurable, and the default is the locale's convention.

**1050. [P0]** As a reviewer, I want an MQM-style review with severities and reasons, so that meaning and fluency problems are found, not only markup.
`UI:` Flags from review: `accuracy` (mistranslation, omission, addition), `fluency`, `terminology`, `style`, `locale-convention`, `layout-fit`; severity (critical / major / minor); the reason and a suggested fix in the drawer; "Apply fix" / "Dismiss with reason"; per-asset review summary card in chat.
`Agent/model:` `loc_review` on task `review.mqm`, a different model from the translator, batched with source, target, terms and context; confidence threshold; back-translation attached for critical flags.
`Pass:` On the seeded-defect fixture (omitted sentence, wrong number, wrong term) all three are found with correct categories; false positives on the clean fixture stay under the configured rate; a `critical` accuracy flag blocks `accepted` until resolved.
`Edge:` Segments under 3 words are reviewed in groups with context; flags on DNT terms are suppressed by preserve rules and counted as such.

**1051. [P1]** As a reviewer who does not read the target, I want back-translation and a plain explanation, so that I can still judge accuracy.
`UI:` "Back-translate" in the drawer shows a literal back-translation with any awkwardness kept, plus a two-line explanation of differences; available in bulk for a filtered view (exported as a bilingual review sheet).
`Agent/model:` `backtranslate` task by a model different from the translator; explanation templated from the diff.
`Pass:` A deliberately wrong number in the target appears wrong in the back-translation; the bulk sheet exports as XLSX with source, target, back-translation and flags.
`Edge:` Idioms are back-translated literally with an "idiom" note, not smoothed.

**1052. [P1]** As a localization manager, I want a consistency audit across an asset and across assets, so that the same button never reads two ways.
`UI:` Chat: "Audit consistency for de-DE" → card listing source terms/phrases translated differently with counts and locations; per row: pick the canonical target and propagate, or mark as intentional; Dashboard shows a consistency score.
`Agent/model:` deterministic grouping by normalized source; LLM only to cluster near-identical sources when enabled (Balanced tier).
`Pass:` Two translations of "Submit" in one asset are listed with both locations; choosing one updates the other with origin `propagated`; intentional differences are remembered.
`Edge:` Case and punctuation variants of the same source are grouped; cross-asset audits respect TM scope.

**1053. [P1]** As a reviewer, I want locale convention checks that know my market, so that dates, numbers, currency and typography are right for the reader.
`UI:` `locale-convention` flags with examples ("Expected 1,24,500.00 for en-IN", "24-hour time for de-DE", "French requires a narrow no-break space before ':'"); a per-locale conventions sheet in Project Settings with overrides.
`Agent/model:` CLDR patterns plus project overrides; deterministic.
`Pass:` `₹124,500.00` in a hi-IN target is flagged with the expected grouping; `09/14/2026` in de-DE is flagged; French punctuation spacing is flagged and fixable in one click.
`Edge:` Segments that legitimately quote a foreign format (a screenshot caption) can be marked "quoted" to suppress the rule.

**1054. [P1]** As a reviewer, I want flags triaged by impact and my dismissals to teach the system, so that I spend time where it matters and see fewer false positives over time.
`UI:` Review tab and Workbench flag list sorted by impact (legal/financial wording > meaning > untranslated visible > terminology > cosmetic) with a short rationale; Dismiss requires a reason from a list ("intentional", "preserve rule", "false positive", "out of scope"); repeated "preserve rule" dismissals propose a preserve rule card.
`Agent/model:` `loc_triage_flags` (Balanced tier) for ranking; dismissal reasons feed preserve rules and per-project QA rule tuning.
`Pass:` The top five flags in the list are the ones a human reviewer ranks highest on the fixture (eval); after accepting a proposed preserve rule the same false positive does not recur.
`Edge:` Dismissals by a user without `loc:review` are recorded as suggestions, not applied.

**1055. [P1]** As a manager, I want a quality scorecard per asset and locale, so that release decisions rest on numbers that move.
`UI:` Dashboard and asset drawer: MQM-lite score 0–10 with dimension bars (accuracy, fluency, terminology, style, locale conventions, layout-fit), flag density per 1k words, trend across versions, and the sample size behind it.
`Agent/model:` scores aggregated from `loc_review` results with severity weights; human-verified flags weigh more than open ones.
`Pass:` Fixing all critical flags raises the score visibly; the score shows "based on N segments reviewed" and drops to "insufficient sample" below a threshold; two locales are comparable on the same scale.
`Edge:` Scores are per asset version; a re-ingest does not inherit the previous score.

**1056. [P0]** As a localization manager, I want QA gate policies, so that nothing exports with known critical problems and regulated content always gets human eyes.
`UI:` Project Settings → Localization → Gates: toggles and thresholds (block export on critical flags; require `loc:review` acceptance for tags `legal`, `finance`, `medical`; require validation pass before delivery; minimum MQM score); the Deliveries tab shows which gate blocks and who can override with break-glass.
`Agent/model:` gates evaluated by `loc_export` and the delivery approval; overrides recorded via `approvals.ts` break-glass.
`Pass:` An export with one open critical placeholder flag is refused with a link to the segment; a `legal`-tagged segment accepted by the agent alone stays "pending human review"; a break-glass override appears in the audit log with the reason.
`Edge:` Gates apply per locale, so a blocked `ar` delivery does not hold `de`.

---

## LH. Visual, in-context and cross-surface validation

**1057. [P0]** As a reviewer, I want a Review tab with source and target screens side by side and AI flags on regions, so that layout and meaning are judged where users see them.
`UI:` Screen navigator strip (thumbnails, status dots), source | target canvas with synchronized zoom/pan, flag overlay with typed markers (truncation, untranslated, overlap, RTL, format), flag list with confidence and actions (Log defect, Fix in Workbench, Dismiss), ⌘E split/single toggle, ⌘B log defect, ↑/↓ screens.
`Agent/model:` screens from `loc_screens`; flags from `vision.lqa` (multimodal best tier, fallback Flash-class); regions in pct-of-media coordinates.
`Pass:` A clipped button on the fixture is marked with a box within the button's bounds; clicking a flag scrolls the segment list to the matching segment when one exists; the tab works for image, video frame, PDF page and document renders.
`Edge:` Screens with multiple target grabs (long pages split) show as one screen with sub-frames; screens with a Figma reference show a third column "design".

**1058. [P0]** As a localization manager, I want the agent to capture the web app per locale, so that in-context review does not depend on someone taking screenshots.
`UI:` Chat: "Capture the checkout flow in ar and de" → Browser tab shows the run; Review tab fills with screens named by page and locale; a coverage list shows pages requested vs captured.
`Agent/model:` `loc_validate` job → `loc-validator` uses `BrowserSession` with `locale`, `Accept-Language`, timezone and the site's locale strategy (path/subdomain/query/cookie); `loc_capture_screen` per state; dedupe by perceptual hash.
`Pass:` For a site with path-prefix locales the `ar` capture visits `/ar/checkout`, not `/checkout` with a header; screens are full-page with the viewport variants configured; the same journey captured twice yields no duplicate screens.
`Edge:` Locale switch via cookie is set before navigation; geo-redirects are detected and reported; auth is reused from saved sessions; a page that fails to load produces a "capture failed" screen with the error rather than a gap.

**1059. [P0]** As a mobile lead, I want per-locale capture on real device sessions, so that RTL, keyboard and truncation on phones are validated.
`UI:` Mobile tab shows the device booting with the locale ("ar_SA · iPhone 15"), the agent walking the journey, and screens appearing in Review; per-device rows in the coverage list.
`Agent/model:` `device_start(language, locale)` and `set_language` from the existing driver; the `mobile-localization` skill's checks; `loc_capture_screen` per screen; native hierarchy used to detect clipped text nodes.
`Pass:` Onboarding → KYC → transfer captured on two devices per locale; the accessibility hierarchy flags a truncated label the vision model also flags (agreement recorded); the run appears in Runs with device evidence.
`Edge:` Apps that need a restart to change language are restarted; system dialogs in the target language are handled by the existing dialog skill; a device pool exhausted parks the job `awaiting_resources` (1098).

**1060. [P1]** As a desktop QA lead, I want per-locale capture of the desktop app on a paired machine, so that installers, menus and dialogs are validated in every language.
`UI:` Desktop tab shows the paired worker, the app relaunching in the target language, and screens per window; menu trees are captured as text for untranslated-string checks.
`Agent/model:` `desktop-driver` app language switch (OS locale or app preference), accessibility tree text extraction, `desktop_screenshot`; `desktop_run_script` oracle for apps that expose state.
`Pass:` Menus and dialogs of the fixture app are captured in `fr-FR`; English strings in the tree are flagged `untranslated` with their window; the run is bounded by `Project.desktop` allowlists.
`Edge:` Apps that need an OS-level locale change ask via HITL before changing the machine; Windows COM vs macOS JXA oracles selected by platform.

**1061. [P1]** As a reviewer, I want to preview translations on the live site before publishing, so that stakeholders sign off in context, not on spreadsheets.
`UI:` Browser tab with an "Overlay: fr-FR" chip; translated text appears in place; hovering shows the segment key; "Report an issue here" logs a defect with the region; shareable review link with expiry for stakeholders (read-only).
`Agent/model:` `loc_preview` injects a locator → text map into the page via the browser driver (only in the agent's browser or the desktop in-app browser; never modifies the site).
`Pass:` Overlaid text matches the Workbench for every resolvable locator; unresolved locators are listed with reasons; the share link shows the same overlay without exposing the workbench.
`Edge:` Sites with CSP blocking injection fall back to a proxied snapshot with a notice; dynamic content re-applies the overlay on DOM changes.

**1062. [P0]** As an e-learning localizer, I want a real SCORM player with a walk-through agent, so that a translated course is proven to run and score.
`UI:` Player tab: course launched via `scorm-again` (1.2 and 2004) in a sandboxed iframe with the overlay locale, TOC/sequencing controls, a cmi trace panel (initialize, set/get value, commit, terminate) and a "Walk course" button; walk results feed the Review tab with one screen per page per locale.
`Agent/model:` `scorm_walk` drives the player in the agent browser following sequencing/navigation events; `scorm_validate` checks manifest schema, resource references and the exported zip.
`Pass:` The golf sample plays in `hi-IN`; completing quiz 2 sets `cmi.score.raw` and `cmi.completion_status` in the trace; the walk visits every SCO and page; an export whose manifest references a missing file fails validation with the path.
`Edge:` SCORM 2004 sequencing with prerequisites is respected by the walk (no skipping); content that calls `API` from nested frames works via cross-frame mode; packages with hardcoded English in JS alerts are flagged as `untranslated` during the walk.

**1063. [P0]** As a marketer, I want a full website locale scan, so that every locale of a published site is checked page by page with an audit of the plumbing.
`UI:` Chat or Assets → Scan: wizard (discovered locales, page selection, viewport), progress by phase (crawl → resolve → capture → analyze → report), a matrix (pages × locales) with status cells, an issues list with severity and regions, and an `hreflang`/canonical/geo-redirect audit section; export PDF/HTML report.
`Agent/model:` `loc_site_scan` job (ported GTW pipeline on `BrowserSession`), `vision.lqa` per screen, deterministic `hreflang` checks; capture concurrency bounded by `BrowsersStatus`.
`Pass:` A site with 20 pages × 5 locales completes with a filled matrix; missing `hreflang` alternates and self-references are listed with the exact tag expected; issues are deduplicated across pages (same header bug reported once with N occurrences).
`Edge:` Locales served only by geo-IP are detected as "not reachable from here" rather than reported as missing; consent banners are dismissed or reported; pages behind login use saved sessions.

**1064. [P0]** As a reviewer, I want layout defects found the way users see them — truncation, overlap, clipping, missing glyphs, wrong mirroring — with evidence, so that a defect is undeniable.
`UI:` Flags carry the region, a zoomed crop, the accessibility node when available, and a pixel diff against the source layout when the layout should match; the defect card shows "expected vs actual" thumbnails.
`Agent/model:` combined signals: accessibility tree (text node bounds exceeding container), `vision.lqa`, `diffPng` region diffs against the source screen, glyph fallback detection (tofu boxes), RTL mirroring rules (navigation, chevrons, progress direction).
`Pass:` On the fixture: a clipped German button, an unmirrored back chevron in Arabic and a missing Devanagari glyph are each found with evidence; the false-positive rate on the clean fixture is below the configured threshold; a defect can be reproduced from the run link.
`Edge:` Intentional non-mirrored elements (logos, media controls) are on a per-project allowlist; screens whose layouts legitimately differ per locale skip pixel diff and keep the other signals.

**1065. [P1]** As a reviewer, I want to draw on a screen and ask the agent about it, so that suspicion becomes a verdict or a defect.
`UI:` Annotation tools (rectangle, arrow, text) on the Review canvas; "Ask about this region" opens a scoped chat thread with the crop; "Locate on source" highlights the corresponding source element; "Compare regions" diffs meaning between source and target areas; annotations have status open/resolved and can link to a defect.
`Agent/model:` `lqa-reviewer` sub-agent with `loc_locate_source`, `loc_compare_regions`, `loc_visual_review` on the crop (multimodal best tier).
`Pass:` Asking "is this truncated?" returns a verdict with confidence and the measured overflow; "Locate on source" finds the element for a translated label; resolving the annotation records who and when.
`Edge:` Regions on video frames carry the timestamp; regions on PDF pages carry the page number.

**1066. [P0]** As a QA lead, I want localization defects to be ordinary defects with a locale, so that one Defects view, one Jira flow and one dashboard cover both halves of the product.
`UI:` Defects view gains facets: locale, flag type, asset, screen; a defect card shows the region crop, the segment link and the locale chip; Jira push adds labels (`loc:ar-AE`, `truncation`), attaches the crop, maps severity to priority and syncs status both ways.
`Agent/model:` `loc_log_defect` writes `issues` with a `loc` facet; Jira via the existing MCP preset with the project's field mapping.
`Pass:` A defect filed from the Review tab appears in Defects next to QA defects and in the Dashboard leakage count; the Jira issue carries labels and the attachment; closing it in Jira updates the defect within the sync interval.
`Edge:` Duplicate detection groups the same defect across locales ("also in fr, ja") as occurrences, not new issues; a Jira project without the custom fields falls back to labels only and says so.

**1067. [P1]** As a reviewer, I want to take over the live surface mid-validation and hand it back, so that I can reach states the agent cannot.
`UI:` "Take control" on the Browser/Mobile/Desktop tab pauses the agent's walk; my interactions are streamed; "Capture this" adds a screen; "Hand back" resumes the job from the current state.
`Agent/model:` existing human-control input APIs; `loc_validate` job checkpoints before pausing.
`Pass:` Taking control during a walk pauses within one action; screens I capture are attributed to me; handing back continues without repeating already-captured screens.
`Edge:` Handing back on a screen outside the planned journey makes the agent re-plan and say so.

---

## LI. Media and documents

**1068. [P0]** As a marketer, I want images with text localized and verified, so that hero art ships in every market without a designer redrawing it.
`UI:` Asset drawer for images shows detected text boxes with the recognized text as segments; after translation a "Render" action shows the localized image beside the source with an OCR readback check (green when the target text reads back exactly); complex images show "Needs source file" with a request card.
`Agent/model:` `image_ocr_layout` (multimodal, structured boxes and style), translate as segments with `maxLength` from box width, `image_localize`: programmatic re-render (`sharp`, font matching) for flat backgrounds, else `image.edit` with the source as `input_references`; `image_verify_text` OCR loop with up to N retries; provenance stored on the artifact.
`Pass:` On the fixture set, flat-background images render with the exact target text and matching font size; edited art passes OCR readback or is flagged; the artifact records model, prompt, references and cost; the original is never overwritten.
`Edge:` Text on curved or perspective surfaces, transparent PNGs, RTL text direction in art, and fonts missing glyphs for the target script (fallback font is chosen and noted).

**1069. [P1]** As a designer, I want the design source used when it exists, so that fidelity is perfect and re-renders are cheap.
`UI:` Assets from Figma/PSD/SVG show "Source available" and render per locale by replacing text layers; the Review tab shows design vs render; a "Push back to Figma" action when the plugin is installed.
`Agent/model:` Figma text nodes via REST and frame renders; PSD/AI text layers via `desktop_run_script` (UXP/JXA) on a paired machine with the app installed; SVG `<text>` replaced directly.
`Pass:` A Figma frame with three text nodes yields three segments and a per-locale render whose text layers changed and nothing else; a PSD on the paired Mac exports per-locale PNGs with layer names preserved; missing Adobe entitlement is reported as "needs entitlement" rather than a failure.
`Edge:` Auto-layout frames that grow with text are re-rendered after layout; text styles with fixed width report `layout-fit` flags.

**1070. [P0]** As a video owner, I want a video transcribed, translated into subtitles that respect reading speed, and exported, so that captions are correct and readable in every market.
`UI:` Video asset shows the transcript as timed segments (speaker, start, end); the Workbench grid shows chars/sec and lines per cue with limits; Player shows the video with the subtitle track and cue boundaries; exports: SRT, VTT, burned-in MP4.
`Agent/model:` `media_transcribe` (`speech.stt` with timestamps and diarization), translate with `maxLength` from reading-speed budgets per locale (e.g. 17 cps for pt-BR, 13 for ja-JP), `media_mux` via `ffmpeg` in the worker.
`Pass:` Cues never exceed the configured cps or two lines; timings are preserved unless a cue is split (with a note); the burned-in export plays with correct glyphs for Devanagari and Arabic (shaping verified visually on a sample frame).
`Edge:` Overlapping speakers, music-only sections (kept as `[music]`), on-screen text that the transcript should not include, and videos longer than the model's input limit (chunked with overlap).

**1071. [P1]** As a marketer, I want dubbed audio per market and a preview I can switch, so that the launch video speaks the language.
`UI:` Player shows track toggles (original, `es-MX` dub, subtitles); per-segment voice and speed indicators; "Regenerate shorter" on cues that overrun.
`Agent/model:` `media_tts` (`speech.tts`) per segment with a per-locale voice table, length fit by regeneration or bounded tempo, loudness normalization, `media_dub` + `media_mux`.
`Pass:` Dubbed cues fit their windows within tolerance; the dub keeps one voice per speaker across the video; the muxed file carries both tracks and correct language tags.
`Edge:` Segments with untranslatable interjections; profanity/legal wording changes require reviewer acceptance before TTS.

**1072. [P2]** As a video owner, I want burned-in on-screen text handled honestly, so that I know what can be fixed without the source project.
`UI:` Detected on-screen text listed as segments with frames; options: "Cover with localized overlay" (preview), "Needs source project", "Leave as is"; generative re-render offered only as an explicit, budget-gated option.
`Agent/model:` frame sampling + `ocr.layout`; overlays via `ffmpeg` drawtext/boxes; `video.generate` only after approval.
`Pass:` Overlays align with the original text position across the shot; the request-for-source card lists the exact timecodes; no generative render runs without an approval card and a recorded cost.
`Edge:` Moving text (lower thirds sliding) uses per-frame tracking or is marked "manual".

**1073. [P1]** As a content owner, I want Office documents merged back with formatting and PDFs handled truthfully, so that deliverables open correctly for the client.
`UI:` Document preview per page/slide with translated runs; export DOCX/PPTX/XLSX; PDF shows the translated DOCX/HTML outputs and a side-by-side review PDF; fonts with missing glyphs are listed.
`Agent/model:` OOXML merge in `loc-formats`; PDF outputs rendered from the translated DOCX/HTML via the existing PDF renderer.
`Pass:` Word opens the exported DOCX without repair prompts; bold/link runs sit on the right words; slide text boxes that overflow after translation are flagged with the slide number; XLSX formulas are untouched.
`Edge:` Text in shapes and SmartArt, headers/footers, footnotes, right-to-left paragraph direction set for Arabic exports.

---

## LJ. Human in the loop: roles, approvals, deliveries

**1074. [P0]** As a localization manager, I want owners per locale and a personal queue, so that everyone knows what is theirs.
`UI:` Project Settings → Localization → Owners: per locale, translators and reviewers (users or "agent"); Workbench filter "My queue" (segments and screens in locales I own, in the states I act on); a badge count on the rail.
`Agent/model:` `Project.localization.owners`; the agent is a valid translator owner; notifications route by owners.
`Pass:` A reviewer for `hi-IN` sees only `hi-IN` items in My queue; assigning a human translator to `de-DE` stops the agent from drafting `de-DE` unless asked; the badge count equals the queue length.
`Edge:` Locales without a reviewer show "no reviewer — deliveries will require an approver" in Deliveries.

**1075. [P0]** As a team, we want a per-locale, per-asset state machine with clear gates, so that "done" means the same thing to everyone.
`UI:` Deliveries tab rows: asset × locale with state chips (translating → in-review → needs-rework → approved → delivered), the blocking gate when any, and actions allowed for my role; state timeline in the drawer.
`Agent/model:` submit for review requires all segments `reviewed|accepted` and all flags addressed; transitions recorded with actor and reason.
`Pass:` "Submit for review" is disabled with the exact reason until the gates pass; a reviewer can request rework with notes; approving moves to `approved` and creates the delivery approval when required; states are visible on mobile.
`Edge:` A new asset version resets the state for changed segments only and shows "partially stale".

**1076. [P0]** As an approver, I want a delivery sign-off card with everything I need, so that I approve in one place with deputies and timers like any other approval.
`UI:` `ApprovalCard` variant `delivery`: asset, locale, coverage, MQM score, open flags by severity, validation result summary, cost, links to Review and Player; Approve / Request changes / Delegate; appears in chat, Approvals inbox and email/Slack link.
`Agent/model:` `approvals.ts` subject `loc_delivery`; `onApprovalFinal` triggers `loc_export` and any configured push.
`Pass:` Approving from the Slack link works with the existing one-tap flow; a timer escalates to the deputy; the export is produced only after final approval; break-glass is logged.
`Edge:` Approvals for many locales at once can be batch-approved with per-locale opt-out.

**1077. [P0]** As anyone, I want the agent's questions to be clear, few and actionable, so that human input unblocks rather than interrupts.
`UI:` Sticky HITL cards above the composer: kinds "question" (1039), "needs source files" (with drop zone), "credentials / sign-in" (existing), "model choice" (two options with cost), "approval to spend"; unanswered cards collapse into a count and live in the Approvals inbox.
`Agent/model:` `request_user_help` with kind and `segmentIds`/`assetIds`; batching similar asks.
`Pass:` No more than one card of each kind is open per asset at a time; answering resumes the job automatically; the card records who answered and when.
`Edge:` Asks during unattended jobs notify owners via configured channels and pause only the affected slice.

**1078. [P1]** As a reviewer, I want a rework loop that fixes only what I flagged, so that review comments become changes without re-translating the file.
`UI:` "Request rework" with notes per segment or per asset; the translator (agent or human) sees a "Rework" filter; the agent's fixes show as drafts with "addresses: your note"; re-QA runs automatically; state returns to in-review.
`Agent/model:` `loc_translate` with `segmentIds` and the reviewer notes in the prompt; `loc_qa` on the touched set.
`Pass:` Only flagged segments change (verified by history); the reviewer sees a diff of each fix against their note; a second rework round is possible without loops (the same note is not re-applied).
`Edge:` Rework notes that conflict with the termbase are surfaced to the reviewer rather than obeyed blindly.

**1079. [P0]** As a localization manager, I want deliveries exported in the right format with a checksum, and pushed to where they live, so that shipping is one approved action.
`UI:` Deliveries: Export (format per asset kind: SCORM zip, strings files in original layout, XLIFF 2.x, TMX for the TM delta, DOCX/PPTX, SRT/VTT/MP4, PNG set), checksum shown, download; Push to a connected target (GitHub PR with files in place, CMS locale entries, LMS upload, TMS hand-off) with a HITL approval card showing exactly what will be written.
`Agent/model:` `loc_export` merges via the filter and validates (`scorm_validate`, XLIFF validation); `loc_delivery_push` via MCP tools; both record artifacts.
`Pass:` The exported strings file differs from the source only in translated values; the SCORM zip passes validation and plays in the Player; the GitHub PR contains the files at the original paths with a descriptive body; nothing is pushed without approval.
`Edge:` Re-export after a change produces a new checksum and keeps the previous artifact; pushes to a target that already has newer content raise a conflict card.

**1080. [P1]** As a team member, I want notifications that match my role and a daily digest, so that I act on time without living in the app.
`UI:` Settings → Notifications gains localization events (assigned, review requested, rework requested, delivery approved, validation failed, budget checkpoint, job done) per channel (in-app, email, Slack, Teams); a daily digest per project; notification cards deep-link to the exact segment/screen/approval.
`Agent/model:` `notify.ts` events; owners from 1074.
`Pass:` A reviewer gets "review requested: hi-IN, 1,284 segments" within a minute of submission; a validation failure notifies the locale owner with the defect count; the digest lists open queues and blocked gates.
`Edge:` Quiet hours per user; deduplication when a burst of segments changes state.

---

## LK. CLI, MCP, CI and connectors

**1081. [P0]** As a developer, I want `teststudios loc push` and `loc pull` in CI, so that strings flow from the repo to the workbench and back without a UI.
`UI:` Terminal output with a summary table (files, segments new/changed, locales) and machine-readable `--json`; the web shows the asset version created "from CI · commit abc123".
`Agent/model:` CLI uses `TESTSTUDIOS_CI_TOKEN` against `POST /api/ci/:project/loc/push|pull`; pull writes files in their original layout and can open a PR via `--pr`.
`Pass:` `loc push ./locales/en.json` creates a new asset version tied to the commit; `loc pull --locales de,ja --pr` opens a PR with only approved translations; exit codes are non-zero on auth or validation errors with clear messages.
`Edge:` Pushing unchanged content is a no-op with exit 0; pulling a locale with a blocked gate writes nothing for that locale and says why.

**1082. [P0]** As a release manager, I want `loc translate`, `loc qa`, `loc pseudo` and `loc validate --fail-on` commands, so that localization quality gates the pipeline like tests do.
`UI:` JUnit and JSON outputs; `--fail-on critical|major`; links to the run and defects in the summary; the Dashboard shows CI-triggered validations with the pipeline id; `loc pseudo` reports overflow and hardcoded strings from the pseudo-locale run (1035).
`Agent/model:` commands enqueue the corresponding jobs and wait (or `--no-wait`); validate uses the configured surfaces and journeys.
`Pass:` A pipeline fails on two critical RTL defects and prints their ids; after the fix the same command passes; JUnit output renders in the CI provider's test tab; `loc pseudo --fail-on major` fails on the fixture's seeded truncation.
`Edge:` Timeouts return a distinct exit code and leave the job running; `--dry-run` prints the estimate without spending.

**1083. [P1]** As a developer, I want the localization tools available to my coding agent via MCP, and the product's engineering sub-agent able to open PRs, so that i18n work happens where code happens.
`UI:` Settings → MCP shows the `loc_*` tools; chat shows `loc-engineer` cards with the diff summary and a HITL "Open PR" approval linking the PR once created.
`Agent/model:` MCP server exposes `loc_*`; `loc-engineer` runs a CLI-agent turn (1029) to externalize strings, wire i18n libraries, add plural handling and resource files, then uses the GitHub MCP to open the PR.
`Pass:` From Cursor, `loc_list_segments` returns the same data as the Workbench; "externalize hardcoded strings in `src/`" yields a PR whose CI passes on the fixture repo; no PR opens without approval.
`Edge:` Monorepos with multiple i18n setups are detected and asked about; generated keys follow the repo's existing naming convention.

**1084. [P1]** As an admin, I want connector presets for design, code, content, TMS and LMS systems, so that assets come from and deliveries go to the tools teams already use.
`UI:` Settings → Connections: presets Figma, Adobe (existing), Canva, GitHub/GitLab, Jira/Linear, Contentful/WordPress/Webflow, Phrase/Lokalise/Crowdin/Smartling (XLIFF hand-off), SCORM Cloud/Moodle; each with scope, test connection and the capabilities it enables (intake, push, both); tokens in the vault.
`Agent/model:` MCP presets in `mcp/presets.ts`; first-party connectors only where no MCP server exists; tools grouped `mcp:<name>`.
`Pass:` Connecting Figma enables the frame picker in Assets → Add; connecting Contentful enables both intake and push with a dry-run preview; a failed test connection shows the precise error and never stores the token.
`Edge:` Rate limits and plan restrictions are shown per connector; connectors can be enabled per project, not only per org.

**1085. [P2]** As a localization manager working with an external LSP or TMS, I want XLIFF round trips and webhooks, so that the platform coexists with the vendor's tools.
`UI:` Deliveries → "Hand off to TMS" exports XLIFF 2.x with states and notes; "Import returned XLIFF" merges by unit id with a diff card; Settings → Webhooks lists events (`loc.asset.ingested`, `loc.delivery.approved`, `loc.validation.failed`) with signed payloads.
`Agent/model:` `typesxliff` round trip; webhook delivery with retries and signatures.
`Pass:` An XLIFF exported, edited in a third-party CAT tool and re-imported updates only the changed units with origin `external`; a webhook consumer receives a signed payload within seconds of approval.
`Edge:` Unit ids the vendor changed are re-matched by source hash with a warning; webhook failures retry with backoff and show in the audit log.

---

## LL. Analytics and dashboards

**1086. [P0]** As a manager, I want a localization dashboard with the numbers that matter, so that status is a glance, not a meeting.
`UI:` Dashboard tab: KPI strip (assets, words in scope, coverage % per locale, MQM score, open localization defects, AI leverage % (MT/TM/human), spend vs cap, estimate accuracy), each with a sparkline and a drill-in; range toggle 7d/30d/90d; orange chart palette; the same `KpiTile`, `Gauge`, `Donut`, `ProgressList` primitives as the QA dashboard.
`Agent/model:` `locDashboard()` aggregates from `loc_*` collections and `usage`; `qa_dashboard`-style tool so the agent can read it.
`Pass:` Numbers match the Assets and Deliveries tabs exactly; drill-in from any tile opens the underlying list filtered; the dashboard loads under two seconds for a project with 100 assets.
`Edge:` Projects with no reviewed segments show "insufficient sample" instead of a misleading score.

**1087. [P0]** As a manager, I want a language quality matrix and a comparison view, so that I see which locales are healthy and which need attention.
`UI:` Heatmap (locales × dimensions: coverage, MQM, open critical, layout defects, validation pass rate) with cell drill-in; a radar comparing locales; sortable by risk; export as image for status reports.
`Agent/model:` aggregates from review, QA and validation results.
`Pass:` A locale with two open critical RTL defects is visibly red on the validation cell; sorting by risk puts it first; the exported image uses the orange palette and legible labels.
`Edge:` Locales added mid-program show "new" until they have data.

**1088. [P0]** As a manager, I want a pipeline funnel and readiness per locale, so that I can promise dates and know what blocks them.
`UI:` Funnel: ingested → translated → QA passed → reviewed → approved → delivered → validated, with counts and words; readiness gauge per locale with blockers listed (gate, defect, approval) and an ETA based on current velocity.
`Agent/model:` readiness formula ported from GTW inputs (review %, flags addressed %, no critical open, no overdue, resolution within SLA) merged with the existing readiness service.
`Pass:` Approving a delivery moves the funnel and the readiness in real time (`live` events); the ETA changes when velocity changes; blockers link to the item that blocks.
`Edge:` Locales with no owner show "no reviewer" as a blocker rather than an ETA.

**1089. [P1]** As a manager, I want predictive and aging views, so that risks surface before deadlines.
`UI:` Defect trend with projection, release outcome prediction per locale, flag-type breakdown (horizontal bars), defect aging buckets, an activity heatmap per locale, and a risk matrix (impact × likelihood) with drill-in.
`Agent/model:` existing `predict.ts` patterns extended with localization signals; deterministic aging.
`Pass:` The projection line updates daily; aging buckets sum to the open defect count; the risk matrix places a locale with rising critical flags in the high cell.
`Edge:` Small samples show wide confidence bands rather than confident lines.

**1090. [P1]** As a finance-minded manager, I want cost and model-efficiency analytics and the ability to compose any chart in chat, so that spend is explainable and questions are answerable.
`UI:` Cost per word by locale and by model, quality per dollar (MQM ÷ cost) by model, spend by task type (translate / review / vision / media), estimate vs actual; chat "Show me cost per word by locale this month" renders a `SpecChart` card; export PDF/HTML report with the orange palette.
`Agent/model:` `compose_analytics` recipes added to `analytics-recipes.ts`; `UsageRow.units` feeds per-word math.
`Pass:` The cost per word for a completed job equals recorded usage divided by translated words; quality per dollar ranks models consistently with the A/B evals; the composed chart matches the dashboard figure for the same filter.
`Edge:` Costs from BYOK keys are shown as "billed to your key" and separated in totals.

---

## LM. Governance, cost, security

**1091. [P0]** As an admin, I want envelopes and checkpoints to cover localization spend, so that no campaign runs past its cap.
`UI:` Governance panel shows localization campaigns alongside QA campaigns; checkpoint cards at 50/80/100% with the option to continue, pause or reduce scope; per-locale tranche caps; the Estimate card shows the remaining envelope.
`Agent/model:` `envelopes.ts`, `campaigns.ts` `recordCampaignSpend` with units; `budgetGate` halts jobs at the cap and parks them `awaiting_approval`.
`Pass:` A job that would exceed the cap pauses at the batch boundary with a card, not mid-batch; approving a boost resumes it; spend attribution is correct per locale and per task type.
`Edge:` Media generation counts toward the same envelope with its own line so a video re-shoot cannot silently consume the translation budget.

**1092. [P0]** As a security officer, I want content confidentiality guaranteed by design, so that client text never leaks across clients or into the wrong provider.
`UI:` Project Settings → Localization → Data: TM scope, "ZDR providers only", "no cross-org TM", retention; a data-flow summary card ("Text goes to: provider X (ZDR). Stored in: TM (project scope), exports. Never stored in: memory."); audit filters for content access.
`Agent/model:` scope enforced in every TM query; provider filter in the router; `guardMemory`; exports and TM deletion on retention.
`Pass:` A cross-project TM search returns nothing; with ZDR on, every job record shows a ZDR provider; deleting a project purges its TM units and artifacts (verified by counts) and leaves an audit entry.
`Edge:` Shared org TM is opt-in per project and visible in the data-flow card; screenshots containing PII are blurred in reports when the policy says so.

**1093. [P0]** As an admin, I want localization capabilities and role presets, so that linguists, reviewers and approvers see and do exactly their part.
`UI:` People → Roles gains `loc:translate`, `loc:review`, `loc:approve`, `loc:manage` in the capability matrix; presets Linguist (translate) and Reviewer (translate + review); route and UI gating consistent (buttons hidden, API refuses).
`Agent/model:` `Capability` union, `ROLE_CAPABILITIES`, `withProjectCap` guards.
`Pass:` A Linguist cannot approve a delivery via UI or API; a Reviewer can request rework; `loc_delivery_push` requires `loc:manage`; denials are audited.
`Edge:` The agent acting as translator inherits the project's policy, not a super role.

**1094. [P0]** As an admin, I want connector tokens and content-source credentials in the vault, so that nothing sensitive sits in settings, TM or exports.
`UI:` Connector setup writes tokens to the vault with health (ok / expiring / expired); UI shows masked values and rotation; exports never include credentials.
`Agent/model:` existing vault records; connectors resolve tokens at call time.
`Pass:` A rotated Figma token takes effect on the next call; an expired token shows in health and in the connector card; grepping exports and TM for token patterns returns nothing.
`Edge:` Site credentials used for capture are the existing saved auth sessions, never typed into prompts.

**1095. [P1]** As an auditor, I want a complete trail from segment to delivery with model provenance and signed exports, so that a delivered file can be traced and trusted.
`UI:` Audit view filters: asset, locale, actor, model, event (accepted, approved, exported, pushed, overridden); a delivery's drawer shows checksum, signer, model summary (which models translated/reviewed what share) and generation ids; generated media list model, prompt and references.
`Agent/model:` audit entries on every state change; export checksum and optional signature; media provenance on artifacts.
`Pass:` For any exported file the trail answers who accepted each segment and which model drafted it; the checksum in the UI matches the file; a break-glass override shows its reason.
`Edge:` Retention deletes content but keeps the audit skeleton (ids, actors, hashes).

---

## LN. Failure modes

**1096. [P0]** As anyone, I want provider outages and model removals handled without duplicate spend or lost work, so that a bad hour upstream is an inconvenience, not a rerun.
`UI:` Progress card shows "provider degraded — switched to fallback" or "paused: no eligible model, awaiting policy change"; job log lists attempts; the `llm` banner explains account/key failures.
`Agent/model:` same-tier fallback list, exponential backoff, idempotency keys per batch, checkpoint before each batch.
`Pass:` Under an injected 30% error rate the job completes with the same result set and no duplicate usage rows; a removed model resolves to the fallback and the job record notes it; a total outage parks the job resumable.
`Edge:` Fallback to a non-ZDR provider is never automatic when the policy forbids it — the job pauses instead.

**1097. [P0]** As a linguist, I want malformed model output caught and repaired, so that a bad completion never lands in the grid or the export.
`UI:` No visible corruption; the job log shows retries; segments the model repeatedly fails on get a `needs-human` flag and appear in the Review queue.
`Agent/model:` schema validation (count, ids, types), placeholder parity, script detection (target in the wrong script), repetition/hallucination checks (target far longer than source, repeated phrases); one automatic re-ask with a stricter prompt, then escalate, then flag.
`Pass:` Injected malformed responses (missing items, extra keys, wrong ids, Latin text for a Hindi target) never write to segments; the flag lists the reason; the batch retry rate is visible in the job summary.
`Edge:` Models that return markdown-wrapped JSON are tolerated; models that ignore the schema three times are demoted in recommendations (1024).

**1098. [P1]** As a release manager, I want validation to wait for resources rather than fail, so that a busy device pool does not turn into red pipelines.
`UI:` Job state `awaiting_resources` with the reason (no Appetize device, Browserless at concurrency limit, paired desktop offline) and an ETA when known; notification to the owner; CI `--no-wait` mode returns a link.
`Agent/model:` `BrowsersStatus` and device pool checks before dispatch; job parks and retries with backoff.
`Pass:` With the Browserless limit reached, the job waits and starts when a slot frees, with no failed screens; the CI command distinguishes "waiting" from "failed" by exit code.
`Edge:` Waiting longer than the configured maximum fails clearly with what was and was not validated.

**1099. [P1]** As a localization manager, I want the source changing mid-job and wrong-language content handled sanely, so that a moving target does not produce a corrupted delivery.
`UI:` When a new version is ingested during a running job, the job finishes on its snapshot and a delta job is proposed; assets whose declared source language does not match the detected language show an amber "Detected fr-FR, declared en-US" card with fix options; targets detected in the wrong language or script are flagged.
`Agent/model:` jobs bind to an asset version; language detection on ingest and on drafts (deterministic script check plus a small model call for close languages).
`Pass:` Two overlapping jobs never write to the same version; the delta job covers exactly the changed segments; a Portuguese file uploaded as English is caught before any translation spend.
`Edge:` Bilingual sources (code comments in English inside a German file) are flagged per segment, not per asset.

---

## LO. Manager and admin altitudes, separate access, legacy safety

*The enterprise shell today is persona-driven (`RAIL` / `LANDING` pages: Mission control and QA floor for managers; Executive summary, Mission control, QA floor and Portfolio for admins) and every aggregate reads `scope.projectIds` from `scopeFor`, which filters only `archived`. These stories add a project-kind dimension once, in scope, and build the localization altitudes on top of the existing page, lens, approvals, live-feed and mobile registries. Nothing here computes its own numbers: every figure at the manager or admin altitude reads the same service the tester-side Dashboard and Deliveries use.*

**1100. [P0]** As a manager or admin, I want a mode filter on the enterprise rail — All · Testing · Localization — applied to every aggregate, so that I can run localization programs from the same control centre without QA noise.
`UI:` Segmented control on the enterprise rail next to the existing scope selector; persisted per persona (`ts_ent_mode_${persona}`) and mirrored in the URL (`?mode=`); hidden when the org has projects of only one kind; in Localization mode the rail takes the orange `data-mode` tokens and the page set changes (Localization floor replaces QA floor; Mission control, Executive summary and Portfolio stay).
`Agent/model:` `scopeFor(userId, orgId, { kind })` and `allowProjects(scope, ids, kind)` gain the filter; enterprise and control routes accept `?kind=qa|localization|all`; panels are untouched.
`Pass:` Switching to Localization changes briefing, board, readiness, cost, timeline and the approvals inbox to loc projects only; reload and URL round-trip keep the mode; a QA-only org never shows the control; `All` payloads are byte-equal to today's responses for the same org.
`Edge:` A user whose scope holds only loc projects lands in Localization mode automatically; a mixed org shows per-mode project counts in the control's tooltip; archived projects stay excluded in every mode.

**1101. [P0]** As a localization manager, I want a Localization floor page, so that programs, quality, readiness, review load and approvals are one screen at the manager altitude.
`UI:` New enterprise page `loc` in the page and lens registries with panels: **Briefing** (loc guidance such as "3 deliveries await approval", "ar-AE blocked by 2 critical RTL defects"), **Programs board** (campaigns of item kind `localization` by state), **Language quality matrix**, **Locale readiness** with blockers, **Review load** per owner (queue depth, turnaround), **Delivery approvals** strip, **Loc job timeline** (ingest · translate · QA · media · validate) and **Team strip** (agent vs human share of accepted segments); each panel drills into the shared right drawer at the exact asset, locale or delivery.
`Agent/model:` `GET /api/orgs/:orgId/loc/floor|readiness|queue?projectId=&kind=` composed from `locDashboard()`, deliveries and owners — the same services the tester-side Dashboard and Deliveries read.
`Pass:` For any project, every floor figure equals the tester-side Dashboard/Deliveries figure (asserted by a test that renders both from one seed); drill-ins land on the exact asset/locale/delivery; the page renders under two seconds for an org with 20 loc projects; live parts are gated by `dashboards:runtime` like the QA floor.
`Edge:` Projects with no reviewed segments show "insufficient sample" rather than a score; the page renders an honest empty state for an org with loc projects but no assets yet.

**1102. [P0]** As an approver, I want delivery approvals in Mission control and on my phone, so that sign-off happens where I already approve campaigns.
`UI:` Approvals inbox rows for subject kind `loc_delivery` show title, summary (asset · locale · coverage · MQM · open critical), cost, links to Review and Player, and Approve / Request changes / Delegate; batch approve with per-locale opt-out; the mobile approval detail screen renders the same subject; Slack, Teams and email one-tap links work as for campaigns.
`Agent/model:` `approvals.ts` subject `loc_delivery` is created with `title`, `summary`, `amountUsd` and `subject.{deliveryId, assetId, locale}`; `onApprovalFinal` triggers `loc_export` and any configured push.
`Pass:` Approve, decline and delegate work from web, phone and a signed one-tap link; the export runs only after the final decision; deputies and timers behave exactly as for campaign tranches; the inbox filter "Deliveries" isolates them.
`Edge:` A delivery whose gates fail again after approval (a new asset version landed) is re-opened with a note instead of being exported.

**1103. [P1]** As an executive, I want a localization block on the Executive summary, so that coverage, quality, spend and readiness across programs are visible at the top altitude.
`UI:` Block "Localization" on the `exec` page: coverage across projects, MQM, spend vs caps with checkpoint marks, readiness per locale with the top blockers; shown only in Localization or All mode; drill-in opens the floor filtered to the project.
`Agent/model:` org-wide roll-up of `locDashboard()` per project in scope; no separate computation.
`Pass:` Block figures equal the sums and means of the per-project floor figures; the block is absent in Testing mode and on QA-only orgs; the drill-in lands on the right project.
`Edge:` BYOK-billed spend is shown as a separate line so caps are not misread.

**1104. [P1]** As an admin, I want the Portfolio to compare localization cost, leverage and quality per dollar per project, so that programs are comparable like QA projects.
`UI:` Mode-gated `portfolio` columns: words in scope, coverage, MQM, cost per word, leverage (TM / MT / human), quality per dollar by model; sortable; CSV export.
`Agent/model:` `usage.units` and review scores rolled up per project; model efficiency from the same evals store the router uses.
`Pass:` Cost per word equals recorded usage divided by translated words for each project; model efficiency ranks consistently with the org's A/B evals; the columns are hidden in Testing mode and on QA-only orgs.
`Edge:` Media-heavy projects show media units separately so cost per word is not distorted.

**1105. [P0]** As an admin, I want a Localization governance page, so that model policy, TM sharing, data policy and audit are managed org-wide from the admin altitude.
`UI:` New admin page `locgov`: org termbases and the TM sharing policy, model policy per task type (org allowlist, ZDR-only, default tiers, per-pair recommendations with their evals), data policy and retention, connector inventory with health, and audit filters (accept · approve · export · override · push) with actor and model.
`Agent/model:` `OrgSettings.localization` and `allowedModels` are read by the router before ranking and by the TM search before scoping.
`Pass:` Blocking a model in policy prevents it on the next job and the job record shows the fallback and reason; toggling TM sharing changes cross-project search results immediately; audit filters return the expected events; the page needs `org:manage`.
`Edge:` Policy changes never rewrite history — existing segment records keep the model that produced them.

**1106. [P1]** As an admin, I want an Organization → Policies screen for the org settings that exist only in schema today plus localization defaults, so that policy is editable without an API call.
`UI:` Tab "Policies" in the People / Organization settings: `allowedModels`, `defaultModels`, `costPolicy`, approval chain and deputies, `defaultSlos`, and the `localization` defaults (default locales, DNT list, expansion budgets, TM sharing, ZDR-only); inline validation; capability `org:manage`.
`Agent/model:` `PATCH /api/orgs/:orgId` settings; enforcement stays where the API already reads each field.
`Pass:` Every field persists and is enforced where it is read today (a model outside the allowlist is refused by the settings patch, cost policy shapes envelopes); an audit entry is written per change; the screen is read-only for non-admins who can see it.
`Edge:` Default locales are a suggestion at project creation, never a constraint on existing projects.

**1107. [P1]** As a localization manager, I want a team and workload panel, so that I can balance review load without rates, hours or invoices.
`UI:` Floor panel "Team & workload": owners per locale, queue depth, median turnaround, acceptance and rework rates, agent vs human share of accepted segments; reassign an owner inline; nothing about pay anywhere.
`Agent/model:` derived from `Project.localization.owners`, segment history and delivery states.
`Pass:` Numbers equal My queue counts and history-derived rates for the same seed; reassigning an owner routes new work and notifies both users; the panel never shows rates, hours or payroll fields.
`Edge:` A user removed from the org disappears from owners and their open queue is flagged "unassigned" on the floor and in the project.

**1108. [P1]** As a manager on the phone, I want localization lenses and a queue widget, so that I can check status and approve without a laptop.
`UI:` Mobile dashboards gain lenses `loc` (floor summary) and `locale-readiness`; home layouts for lead, pm and super_admin gain a "Localization queue" widget (count plus top blockers); the project list shows the kind badge; deep links from notifications open the right lens.
`Agent/model:` the same `GET /api/orgs/:orgId/loc/*` payloads as the web floor.
`Pass:` The `loc` lens renders for pm and super_admin; the widget count equals the web My queue; delivery approval from the phone behaves like other approvals when the connection drops; orgs without loc projects hide the widget and lenses.
`Edge:` Unknown lens ids from an older app build fall back to the default dashboard rather than crashing.

**1109. [P0]** As any user, I want deep links and entry points that respect the workflow, so that a delivery link never lands me on the Runs tab.
`UI:` `/approvals/:id` routes by subject kind — a delivery opens the Deliveries tab with the row focused in the tester shell, or the Mission inbox for managers and admins; `?mode=localization&page=loc|locgov` opens the enterprise pages with the orange rail; the org home shows two entry points, "Test something" and "Localize something".
`Agent/model:` none.
`Pass:` A delivery link opens Deliveries with the row focused; campaign links keep today's behaviour; deep links open in the right mode and page; both entry points create or select the right project kind.
`Edge:` A link to a project outside the user's scope shows the existing "no access" state, not a blank shell.

**1110. [P1]** As an admin, I want role presets and landing rules, so that linguists, reviewers and localization managers start in the right place.
`UI:` People → Roles offers presets Linguist (`loc:translate`), Reviewer (`loc:translate` + `loc:review`) and Localization manager (manager capabilities + `loc:manage`); invites with `projectRoles` on a loc project land the invitee on that project's My queue; a manager whose scope is loc-only lands on the Localization floor.
`Agent/model:` the existing custom-role mechanism; landing computed from scope and project kinds; the People capability matrix is derived from the API, so the new capabilities appear without UI changes.
`Pass:` A Linguist cannot approve via UI or API; a Localization manager lands on the floor; the invite flow lands on My queue of the invited project; presets show in the capability matrix.
`Edge:` A user with both QA and loc roles keeps the persona toggle and the mode control; landing follows the last-used mode.

**1111. [P1]** As a localization-only organisation, I want mode-aware copy, so that nothing on screen talks about testers, smoke tests or the QA floor.
`UI:` One module (`modeCopy(mode, key)`) supplies the strings for the enterprise rail and page titles, briefing guidance, board, team strip, timeline, run drawer, quality, org map and ROI panels, the device live view, and the mobile registry and dashboards; ternaries scattered across components are not allowed.
`Agent/model:` none.
`Pass:` On a loc-only org a denylist grep in the spec finds no QA-only copy on any enterprise page; a QA-only org matches its screenshot baselines; a mixed org in All mode uses neutral copy ("Projects", "Programs").
`Edge:` Third-party strings (Jira status names, model names) are never rewritten.

**1112. [P2]** As a manager, I want a locale monitor kind, so that published sites are re-scanned per locale on a schedule with alerts.
`UI:` Monitors panel: kind `locale-scan` with URL, locales and cadence; check history with matrix deltas; alerts into Mission control and Slack; disabling stops runs.
`Agent/model:` the scheduler enqueues a site-scan `loc_validate` job per cadence; alerts fire once per new issue or hreflang regression.
`Pass:` A monitor runs on schedule, records each check, alerts once per new issue and never for known ones; concurrency is shared with ad-hoc scans (waits rather than fails).
`Edge:` A monitored site that starts redirecting by geo-IP is reported as "unreachable from here", not as a regression.

**1113. [P1]** As anyone, I want org-wide search to include localization content, so that a phrase in any language finds its segment.
`UI:` The Search panel indexes loc assets, segments (source and every target), terms and deliveries; a segment result opens the Workbench at that row, a delivery result opens the Deliveries row.
`Agent/model:` text indexes on `loc_segments` source and per-locale targets; results are scope-aware and honour TM scope.
`Pass:` Searching a target phrase finds the segment and opens the Workbench at it; results never include projects outside the user's scope; results are grouped by kind.
`Edge:` Queries under three characters require a locale or asset filter to avoid flooding.

**1114. [P0]** As an existing customer, I want everything I have today untouched, so that localization mode is purely additive.
`UI:` QA-only orgs see no mode control, no loc pages and no orange; the tester and enterprise shells are pixel-identical to the pre-change baselines in both themes; persisted tab strips and pane layouts load unchanged.
`Agent/model:` projects without `kind` read as `qa`; `publicProject()` only adds fields; `scopeFor` without a kind filter returns identical results; `selectSkills` for a QA project returns the same set as before.
`Pass:` Baseline screenshots (tester dark and light, Mission control, QA floor, Executive summary, Portfolio) match on a QA-only org; a stored `ts_workspace_open_tabs_v2` value from before the change loads through `readOpenTabs`; the full existing API and e2e suites stay green; API p95 for project listing does not regress with the new `(orgId, kind)` index.
`Edge:` A QA project that later opens loc tabs from More gains them without changing its kind or theme.
