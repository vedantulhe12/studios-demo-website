# Accessibility

KaDeep checks content and products for accessibility: uploaded files (PDF, Word, PowerPoint, Excel, EPUB, InDesign IDML / InCopy ICML, images, video and audio, websites and SCORM packages), in their original language and in every translation, and live websites, mobile apps and desktop apps. Results read against WCAG 2.2 (A / AA / AAA) and the laws that cite it.

## Standards and law lenses

`packages/shared/src/a11y-standards.ts` holds every WCAG 2.2 success criterion (plus 4.1.1 Parsing, still cited by EN 301 549 and Section 508) and the lenses:

| Lens | Basis | WCAG scope | Clause numbers |
| --- | --- | --- | --- |
| WCAG 2.2 | W3C | 2.2 at the chosen level | SC id |
| EN 301 549 | v3.2.1 | 2.1 AA | 9 (web), 10 (documents), 11 (software) + SC |
| European Accessibility Act | Directive (EU) 2019/882 via EN 301 549 | 2.1 AA | EN clause |
| Section 508 | 36 CFR 1194, E205 / E207 | 2.0 AA | E205.4 / E207.2 + SC |
| ADA Title II | 28 CFR 35.200 (2024) | 2.1 AA | 28 CFR 35.200 + SC |
| Accessible Canada Act | CAN/ASC-EN 301 549:2024 | 2.1 AA | CAN/ASC-EN clause |

The same findings can be re-read under any lens (`GET /a11y/audits/:id/lens/:law`). PDF findings also cite PDF/UA (Matterhorn) checkpoints and EPUB findings EPUB Accessibility 1.1 requirements.

## Engines

| Engine | Content | What it decides |
| --- | --- | --- |
| `office` | DOCX, PPTX, XLSX | title, language, picture / chart alt text, headings, table header rows, link text, text contrast, slide titles and reading order, sheet names, hidden sheets, merged cells |
| `pdf` | PDF | tagged, /Lang, title and DisplayDocTitle, figure alt (boxed on the picture), headings, table headers, form field names, bookmarks, scanned pages, PDF/UA claim; tags lost by a translation |
| `epub` | EPUB | dc:language, accessibility metadata, conformance claim, navigation, page list, then every chapter as HTML |
| `indesign` | IDML, ICML (`packages/loc-formats` `readIdml`) | what InDesign carries into its tagged PDF / EPUB export: title (XMP), text language (also a translation that kept the source language), alt text on placed pictures (Object Export Options; artifacts skipped, XMP-sourced alt flagged for review), H1–H6 export tags on heading styles, table header rows, the Articles panel as reading order (review). A binary `.indd` is refused with the steps to save it as IDML |
| `html` | HTML pages in assets (Markdown is rendered first) | lang, title, alt, labels, names, link text, headings, tables, frames, zoom, captions, autoplay |
| `image` | pictures | text in the image (from the asset's OCR) and its measured contrast |
| `media` | video, audio | captions per language (also for dubbed languages), transcript, audio description (review) |
| web / device / desktop | live URL, app screens, desktop app | axe (violations and needs-review), keyboard order and focus, reflow, contrast from screenshots, screen-reader transcript |

What an engine cannot decide is listed as a manual check; untested criteria stay "manual" in the criteria table and are never counted as passes.

## Source and translation

"Check both & compare" audits the original and a translation as a pair. A finding in the translation whose fingerprint (rule + place) is not in the original is marked **introduced by translation** — the usual ones are a language setting left as the source language, untranslated alt text, a dubbed video without captions, and PDF tags lost when the translated PDF is produced.

## Score and verdict

Each open failure costs by impact (critical 15, serious 9, moderate 4, minor 1.5; repeats add a little; review items a quarter). Any open critical or serious failure means **does not conform** (score at most 79); only moderate / minor failures mean **partially conforms** (at most 89). When nothing failed but no criterion was decided either (a picture whose text was never read), the verdict is **not evaluated** and the UI shows no score: an unchecked file never reads as conforming. A zip with no HTML or Markdown pages is refused instead of passing with zero pages.

## Fixes

Every finding has steps; the safe mechanical ones are applied to a copy (`POST /a11y/audits/:id/fix`), never the original: document language, title, PDF title bar, Word table header rows, EPUB accessibility metadata, and alt text the user approves. "Save as new version" keeps the fixed original as the asset's next version. Tagging a PDF, reading order and contrast stay as guided steps.

**AI help with alt text** (needs an OpenRouter key; every call goes through the router's `vision.lqa` task and the usage ledger). "Suggest alt text" (`POST /a11y/findings/:id/suggest-alt`, `a11y/ai.ts`) drafts wording in the audit's language for an Office, EPUB, package or image picture, and for a PDF figure: the page is drawn with pdf.js (its @napi-rs/canvas backend), the figure's box cropped with pngjs and sent with the page text nearest it (`a11y/pdf-figures.ts`). An approved PDF draft is written as `/Alt` on the page's Figure structure elements without alt, in structure-tree order (`altText[findingId]` for the first, `altText["<id>:2"]` for the second); an untagged PDF is told to make an accessible copy first. `ai: true` on an audit adds a review of alt text that already exists: in files (`engines/vision.ts`) and, for url audits, on the live page after axe (`engines/vision-web.ts`: each `<img alt>`, `input[type=image]` and `role="img"` with aria-label, up to 20 per page, element screenshot, once per page not per view). Findings are `vision-alt-quality` / `vision-long-description`, outcome review, titled AI-assessed. No key means a manual item, never a failed audit. Tests: `a11y-ai.test.ts`, `a11y-vision-browser-depth.test.ts`.

**Accessible copy of an untagged PDF** (`{ remediate: 'accessible-pdf' }` on the same route, `a11y_fix remediate`, "Make an accessible copy" in the panel; `a11y/pdf-remediate.ts`, `a11y/accessible-copy.ts`). The text is read with pdf.js and rebuilt as semantic HTML: reading order per page (two-column pages left column first), headings ranked by font size (H1–H3, never skipping a level), lists from bullet / number patterns, simple tables where cells align (first row as `th`), embedded pictures with alt text from the request (keys `p<page>-img<n>`; empty = decorative) or the original's `/Alt`, one section per original page at its size. Chromium prints it with `tagged` and `outline` (bookmarks from headings); pdf-lib then sets `/Lang` (audit locale), the title and DisplayDocTitle, and adds the PDF/UA identifier only when the re-check (`checkPdf` on the copy) has no failing machine check and no picture lacks alt text. The response lists what was done, pictures still needing alt text, the remaining findings and the resolved rules; the copy downloads as `<name>-accessible.pdf` (`GET /a11y/audits/:id/accessible`) or is kept as a version. The layout is re-flowed and differs from the original, which the response and UI say; prefer Acrobat remediation when the look must be kept. A PDF with no text layer is refused (OCR first). Tests: `a11y-pdf-remediate.test.ts` (browser-free model and HTML), `a11y-pdf-remediate-browser-depth.test.ts` (print and re-check).

## Where it shows

- InDesign assets preview as a **Story view** (`GET /loc/assets/:id/stories?locale=`): stories in order, each paragraph at the level its style exports to, pictures with their alt text, tables, reading order; findings are marked on the story, picture, table or style they are about.
- An asset's **Accessibility** tab, in three parts: **Results** (score, criteria, findings, fixes; the full-size view numbers the findings on the file), **Confidence** and **Reports** (both below).
- Reports (audit report and Accessibility Conformance Report) and the Accessibility dashboard.
- Reports: see [Reports](#reports) below; the localization delivery report (`POST /loc/deliveries/:id/report`) carries quality and source-vs-translation accessibility scores.
- Live checks (`apps/api/src/a11y/live.ts`, engines `web` / `device` / `desktop`): axe plus keyboard walk, reflow, text spacing, zoom and reduced motion per viewport and colour scheme. The keyboard walk also opens up to five dialog triggers with Enter (never links to other pages) and checks focus moves in, Tab stays inside a modal, Escape closes it and focus returns (2.1.2, 2.4.3, 2.4.11, 3.2.1). App screens get screenshot contrast and a screen-reader transcript, a large-text pass (`largeText`, default on: 150% and 200% font compared with the current size for clipping, truncation, overlap and controls pushed off-screen, 1.4.4, boxed on the 200% screenshot, original scale restored; a manual item where the lane cannot change Dynamic Type) and contrast per requested colour scheme (`location.scheme`, original appearance restored). Desktop windows are checked from the accessibility tree; whether frames are screen- or window-based is decided per snapshot and its confidence is stored on each finding (`frameSpace`, `frameConfidence`). Evidence (screenshots, annotated copies, crops) at `GET /a11y/evidence/:auditId/:name`.
- Dashboard (`GET /a11y/dashboard`, org roll-up `GET /api/orgs/:orgId/a11y/dashboard`): the Accessibility dashboard tab, a card on the localization dashboard, and the Accessibility panel for testing projects.
- Whole-site checks (`apps/api/src/a11y/site.ts`, subject `{ kind: 'site', urls? | sitemap? | fromRuns? | startUrl?, maxPages?, sameOrigin: true }`): the pages come from a list, a sitemap (a sitemap index is followed, up to 20 files; compressed sitemaps are a manual item), the pages the project's web runs visited (`Run.actions[].url` and `finalUrl`, deduplicated by path, on the base origin), or a same-origin breadth-first crawl from a start page that never follows sign-out / delete / unsubscribe links, files, other origins, downloads, nofollow or hash-only links. `maxPages` defaults to 25, hard cap 200. Each page gets the web engine (and the AI alt-text review with `ai`, first view only) in one browser session per view, a fresh navigation per page; the job shows "page 7 of 25" and each finished page is checkpointed (`audit.site.pages`, and its result in the object store), so a re-run carries on from the next page. Findings are per page (keys carry the path, labels the page title and path, `location.url` the page); units are pages, `coverage.examined` the page URLs. `GET /a11y/audits/:id` adds `pages` (score per page, worst first); the panel's "Pages" selector picks the source and the results get a per-page score table, a page filter and group by page; the report adds a Pages section (score table, heat strip by page, problems repeated across pages); the dashboard shows the site with its page count. Tests: `a11y-site.test.ts`, `a11y-site-ai.test.ts`, `a11y-site-browser-depth.test.ts`.
- Chat: `a11y_audit`, `a11y_findings`, `a11y_fix`, `a11y_report`, `a11y_confidence`, `loc_delivery_report` (skill `accessibility-audit`).

## Reports

`apps/api/src/a11y/report-document.ts` builds the client report, which is the default. It is a `ReportSpec` with `style: 'document'`, rendered by `services/report-doc.ts` in the look of the KaDeep UAT review. On screen it is a stack of A4 sheets that reflows to the width of a phone. When printed, the page box sets the margins, a running header and page numbers (none on the cover), and the PDF comes out **tagged with bookmarks**, so the accessibility report is itself accessible. The cover carries the KaDeep Studio Pro and Kadeep.ai marks (`services/report-brand.ts`, embedded so printing needs no network).

| Section id | Page |
| --- | --- |
| `summary` | Verdict under the law, score, failures, conformance by WCAG principle, findings by impact, priority risks, what passes |
| `method` | Open › Check › Compare › Report › Verify with the real counts, engines applied, safeguards, scope |
| `conformance` | Every criterion in scope (with the law's clause numbers), and the ones that fail or need review |
| `violations` | The violation matrix (criterion × page, occurrences, coloured by worst impact), heat strip, most violated criteria |
| `pages` | Site audits: score per page and problems repeated across pages |
| `translation` | A source / translation pair: what the translation fixed, carried over and introduced |
| `findings` | Severity mix, by page and by check, then each critical / serious / moderate failure with Expected / Steps to fix / Why, evidence, criteria, clause, where, effort |
| `all-findings` | Every open finding with its phase |
| `confidence` | How sure we are (below), with a reviewer sign-off block |
| `manual` | Engine call-outs, the screen reader pass, criteria for a manual check |
| `plan` | Fix now / next / later, the five-step plan, what we need from the client, the remediation table |
| `appendix` | Scoring, all criteria with ACR remarks, closed findings |

`kind: 'acr'` is the VPAT 2.5 Accessibility Conformance Report in the same style. It has a table per WCAG level and a "How this report was verified" page. Options are the same over HTTP (`POST /a11y/audits/:id/report`, `GET …/report.html?…`) and in chat (`a11y_report`):

- `sections` keeps only some pages, by id; `omit` drops some;
- `title`, `subtitle`, `coverLead`, `preparedFor` (text only; the marks stay KaDeep's) and `intro` (replaces the generated summary paragraph) change the wording;
- `extraSections` adds custom pages built from report blocks, placed before the appendix;
- `style: 'standard'` gives the earlier infographic layout.

Saved reports carry `meta.a11y` (audit, asset, locale, kind, law, score, customised), and `GET /a11y/reports?assetId=` lists them. That list is the asset's **Reports** tab, which also offers a choice of report, standard, cover wording and pages; an inline A4 preview; PDF download or a web page; and "Customize with the agent", which puts an editable request in the chat composer (`ts:compose`). `create_report` also takes `style: 'document'`, with the blocks `bar-list`, `process`, `chips`, `segment-bar`, `violation-matrix`, `risk-list` and `half: true` to pair two blocks on a row. That lets the agent build any other client report in the same template. Tests: `a11y-report.test.ts`, `a11y-confidence.test.ts`, e2e `zz-loc-a11y-kgl.spec.ts`.

## Confidence

The question a compliance buyer asks is "how sure are you?". `apps/api/src/a11y/confidence.ts` (`GET /a11y/audits/:id/confidence`, the **Confidence** tab, `a11y_confidence`) answers it from measured data only:

- **How each criterion was decided.** Decided by an engine (pass or fail from a deterministic rule), AI-assisted (only the vision review touched it), flagged for a person, checked by a person (a reviewer ticked it in the sign-off), still manual, or not applicable. The share an engine decided is reported as-is. It is often low for documents, and that is the honest number.
- **Seeded-defect benchmark** (`a11y/calibration.ts`, `GET|POST /a11y/calibration`, stored in `a11y_calibrations` with the engine versions). This follows the method KGL's experts use to qualify a checker. A clean control file is built for each format (web page, PDF, Word, EPUB, InDesign), then one variant per known defect with exactly that defect added (35 today, InDesign included), and every file goes through the same engine an audit uses. A seed counts as detected when one of the rules named for it is reported, and any failure on a clean control is a false positive. Recall carries a Wilson 95% lower bound. Three more seeds are judgement calls (alt text that describes the wrong picture, instructions by colour alone, PDF tags in the wrong reading order). They are expected misses, kept out of the recall, and shown to make clear where a person is needed.
- **Agreement.** Across the project, findings people closed versus findings they rejected as not an issue, plus each sign-off's confirmed / rejected / missed counts.
- **Human sample.** A fixed share of the project's checked files (default 10%, `?rate=`; KGL samples 5–10%) is chosen by hash, so the same files stay selected. Any file whose check claims to conform is added. A reviewer signs a check off with `POST /a11y/audits/:id/review` `{ verdict: 'agree'|'changes', confirmed, rejected, missed, criteria?, notes? }`. The sign-off appears in the report's confidence page and in the ACR's disclaimer. `GET /a11y/sample` lists the sample and what is reviewed.

## Scheduled checks

An automation of kind `a11y` (`Automation.kind`, config `Automation.a11y`: `law`, `level`, and what to check) audits on its schedule (`apps/api/src/a11y/schedule.ts`). Testing projects check the project's address, a list of pages (`urls`), the whole site (`site: true`, a `site` subject from the address) or any subjects given as they are (`subjects`). Localization projects check every visual asset (the latest version of each document, PDF, EPUB, picture, video, site or course; strings files have no pages and are skipped) or the assets named in `assetIds`: the original first, then each translation linked to it, so every translation is compared (`compare: false` checks originals only). The scheduler queues the audits as one `a11y_audit` job carrying the automation id; no model key is needed.

After the run each finished audit is compared with the previous finished audit of the same subject and level (`A11yAudit.change`: previous score, score delta, new critical / serious failures by fingerprint). A lower score or a new critical or serious failure is a **regression**: the project owner gets a bell notification (`a11y.regression`, also emailed) and the project's Slack / Teams (or `NOTIFY_WEBHOOK_URL`) gets the same text; the automation's last status becomes failed. Every run adds a point to `Automation.a11yTrend` (average score, critical and serious counts, regressed); the audits themselves feed the dashboard's trend.

Create one on the Automations page ("Accessibility check": daily or weekly, standard, level, pages or the whole site), with `POST /api/projects/:id/automations` `{ kind: 'a11y', a11y, trigger }`, or with `POST /api/projects/:id/a11y/schedule` `{ cadence: 'daily' | 'weekly', law?, level?, urls?, site?, assetIds?, gate? }`, which creates or updates the project's one check. The Project Charter calls the latter: a testing project with "Accessibility" among its test types gets a weekly check of its address; a localization project with "Check accessibility of every delivery" gets a weekly check of every asset and `gate: true`, which turns on the delivery gate (`LocGates.noNewA11yFailures`). Tests: `apps/api/test/a11y-schedule.test.ts`.

## CI

With the project's CI token (Settings, or the charter's Connections step), the same auth as `POST /api/ci/:projectId/run` (`apps/api/src/routes/a11y-ci.ts`):

- `POST /api/ci/:projectId/a11y` `{ url? | urls? | site? | assetId? (locale?, compare?), law?, level?, failOn?: 'critical' | 'serious' | 'moderate', minScore? }` starts the audit (no subject: the project's address). It answers 202 with `auditId`, `jobId`, `statusUrl`, and `audits` when several pages were given.
- `GET /api/ci/:projectId/a11y/:auditId` → `{ status, done, score, grade, verdict, counts, failed, gate, reportUrl }`. `failed` is true when an open failure is at or above `failOn` (default `serious`), the score is below `minScore`, or the audit could not run; `failOn` / `minScore` in the query override the ones stored with the audit.
- `wait=1` (on either) holds the answer until the audit finishes, up to `timeout` seconds (default 120, at most 900; `wait=<n>` waits up to n). A finished answer that fails the gate is **422**, so `curl --fail-with-body` fails the step.
- `GET /api/ci/:projectId/a11y/:auditId/report.html` is the audit report, with the same token.

GitHub Actions, after the deploy step (secrets `KADEEP_API`, `KADEEP_PROJECT`, `KADEEP_CI_TOKEN`):

```yaml
- name: Accessibility gate
  env:
    API: ${{ secrets.KADEEP_API }}
    PROJECT: ${{ secrets.KADEEP_PROJECT }}
    TOKEN: ${{ secrets.KADEEP_CI_TOKEN }}
  shell: bash
  run: |
    code=$(curl -sS -o a11y.json -w '%{http_code}' -X POST \
      -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
      -d '{"url":"https://staging.example.com","failOn":"serious","minScore":80}' \
      "$API/api/ci/$PROJECT/a11y?wait=1&timeout=600")
    jq -r '"Score \(.score) (\(.verdict)), failed: \(.failed) — report: \(.reportUrl)"' a11y.json
    test "$code" = 200
```

Tests: `apps/api/test/a11y-ci.test.ts`.
