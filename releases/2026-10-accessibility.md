# Release notes: Accessibility Studio, real file previews, new project setup (dev, October 2026)

## Accessibility, end to end
- **Check anything.** Uploaded files (PDF, Word, PowerPoint, Excel, EPUB, images, video and audio, websites, SCORM), live web pages and whole sites (a list, a sitemap, a crawl, or the pages your test runs visit), mobile app screens and desktop apps.
- **Original and translation, side by side.** Check both and the problems the translation introduced are marked, such as a language setting left unchanged, untranslated alt text, or captions missing in a dubbed language.
- **Any standard.** WCAG 2.2 at A, AA or AAA, read under EN 301 549 / the European Accessibility Act, Section 508, ADA Title II or the Accessible Canada Act. PDFs also cite PDF/UA and EPUBs cite EPUB Accessibility 1.1.
- **Findings you can act on.** Each finding says what is wrong, why it matters, the criterion and the clause of the law. It is numbered on the file itself (PDF, Word, slides, sheets, EPUB chapters, web pages) and has evidence screenshots and step-by-step fixes.
- **Fixes:**
  - Safe fixes are applied to a copy: language, title, table headers, EPUB accessibility metadata, and alt text you approve.
  - Untagged PDFs can be rebuilt as a tagged, accessible copy.
  - AI drafts alt text from the picture and its context, in the file's language.
- **Live checks:**
  - **Web:** keyboard and dialog walks, reflow, zoom, and dark mode.
  - **Mobile:** large text, contrast in light and dark mode, and what a screen reader would announce.
- **Reports:** an infographic audit report and a VPAT-style Accessibility Conformance Report, each under the law you choose. Localization deliveries get an infographic delivery report.
- **Tracking:**
  - An **Accessibility dashboard** with score, trend, conformance per law, top failing criteria, and original vs translation.
  - **Scheduled checks** that alert you when the score drops.
  - A **CI endpoint** that can fail a build.
  - A delivery gate that blocks translations which introduce new serious failures.
- **From chat:** the agent runs checks, explains findings, drafts and applies fixes, and produces reports, with a live card in chat.

## See the real file
- Click a PDF, Word, PowerPoint, Excel or EPUB asset to see the file itself (pages, slides, sheets or chapters), in the original or any language, side by side at full size. The Workbench preview can show the file too.
- EPUB is a supported format.
- Translated PDFs in Hindi, Arabic and other shaped scripts now draw real glyphs; before, they were delivered as "????".

## New project setup and navigation
- A **Projects** page after sign-in, with Settings available before you open a project.
- **Project Charter** setup in five steps: the project, a platform matrix by browser and device, connections tested live, rules, and a review. Choosing Accessibility schedules weekly checks.
- Spotlight search (⌘K), a movable agent dock, notifications in the sidebar, and localization projects using the same sidebar as testing projects.

## Fixes
- Row actions in the assets list stay clickable in narrow windows.
- Localization colours no longer carry over to the Projects page or to testing projects.

See `docs/ACCESSIBILITY.md` for the full guide, including the CI snippet.
