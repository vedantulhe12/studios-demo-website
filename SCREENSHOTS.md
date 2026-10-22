# Saved screenshots, full pages, comparison and annotated defects

The agent can keep screenshots as files, capture a whole scrolling page, compare two screenshots of the same screen
(two languages, two builds, before / after) and file defects whose evidence is the exact screenshot with numbered
marks on the problem. Everything works on every device lane (Appetize, emulator / simulator, real phone), on web pages
(the page's text is read in the browser) and on desktop app windows (the window's accessibility elements). Screenshots
of the same screens in several languages go side by side into Localization Review → Translations. Nothing in the logic depends on a particular app: placement, pairing and checks come
from the screen's own element map, the pixels, project settings and the project's do-not-translate lists.

## Tools

| Tool | What it does |
|---|---|
| `capture_screen` | Saves a full-resolution PNG named by the system: `<language>/<NN>-<screen>.png` in the chat's set. NN is fixed the first time a screen is captured, so every language sorts the same. The same screen + language replaces the file (same artifact and link); the earlier one stays as a step picture of its run. `full_page=true` stitches a scrolling page. Captures the device, the page or a desktop window (`surface`, default in that order). A screen name that is new for this language but looks like another language's screen (by layout) gets a one-line hint to reuse that name. |
| `list_captures` | The chat's (or project's) saved screenshots with paths and capture ids. `coverage=true`: per language, the original language's screens it still lacks, the ones captured under another name and the extra ones. |
| `loc_review_from_captures` | Localization: pairs the project's screenshots by screen and puts them in Review → Translations (see below). |
| `annotate_capture` | Draws numbered red boxes on a copy, placed by element text / ref (exact boxes from the element map) or a percent box; `show_elements=true` lists the element map. Saved as `annotated/<NN>-<screen>-<language>.png`. |
| `compare_captures` | Pairs the two element maps and lists candidates; saves a side-by-side picture (A left, B right, candidates numbered on both) and shows it to the agent. `kind=localization` or `visual`. |
| `file_issue` (`evidence`) | Builds the annotated picture, the original and a close-up per mark for each named screenshot; required when the chat has saved screenshots. |
| `export_captures` | The chat's set as one zip in the chat (same layout as below). |
| `join_capture` | A long screenshot saved in parts: shows its doubtful joins as lettered strips, joins it at the picks (the agent's must match at the cut; `chosen_by=user` for the user's), or asks the user (`ask_user=true`). |
| `device_screenshot` | Unchanged: looks only, saves nothing. |

## Capture quality

- Always the device's own PNG (`screenshotPng()`), never the live-view JPEG or a step picture.
- Before every capture: animations settle, loaders / progress bars / toasts disappear (up to 8 s), the keyboard is closed with the keyboard toggle (never BACK). Anything still showing is reported.
- A black frame while the element tree has text = a protected (secure) screen: reported, not saved.
- Status bar: cropped by default on real phones (height from the system insets, else 24 dp × density; iOS from the screen size's safe area), optional elsewhere; element boxes move with it.
- iOS boxes are scaled from points to pixels by image width ÷ root element width.
- Web pages: one script in the page reads the visible text-bearing elements (role, `data-testid` / `id` / `name` of the element or its link / button, box); boxes are in page coordinates for a full page and scaled by screenshot width ÷ page width (device pixel ratio). A page that blocks the script is saved without a map.
- Desktop windows: the window's accessibility elements with a label or value; frames in screen or window coordinates (the space more frames fit in; screen on a tie), scaled to the picture (Retina). A picture of the whole display instead of the window gets no map rather than wrong boxes. No full page: each scrolled view is its own screen. A driver that returns JPEG is refused (screenshots are PNG).

## Full page (`services/stitch.ts`)

1. Scrolls with the device's own scroll where the lane has it (`scrollBy`: agent-device `scroll` on real phones — moves at once, damps release momentum, reports `moved` / `at-edge` / `unchanged`), otherwise a timed swipe from inside the scroll area. Never `drag`: that is press-and-move, and on a link or card it becomes a long press (a context menu) instead of a scroll.
2. Each scroll is quick (≤ 450 ms, about half the scroll area): a slow injected gesture moves in steps smaller than the touch slop, so the view under the finger sees a press and opens its long-press menu (seen on a real phone at 1.2 s, never at ≤ 0.5 s).
3. Each step's scroll is measured from elements seen in both screen reads; pixel rows confirm it. Partly hidden elements (trimmed bounds, their height changes) are not evidence. Large pictures (photos, video, maps — parallax or late-loading) are left out of the pixel check; three or more elements moving by the same distance outweigh pixels that disagree in places. When the elements' answer is not borne out, an unmistakable pixel match wins.
4. The fixed header and bottom bar are kept once (never beyond the first / last scrolling element). A bar may hold a small changing part (status-bar clock, a badge); a bar that changes as a whole (collapsing / recolouring header) makes the page come out as numbered parts.
5. Floating overlays (a floating button, chat bubble, a system edge-panel handle) stay put while the page scrolls; each page row is taken from a screen where it is not under one, so they appear once instead of on every screen. Elements of other apps (by resource package) are kept out of comparisons.
6. Stops at the end of the page (no movement, or the device says so), at the segment cap (default 8, up to 15) or the time limit (`CAPTURE_DEADLINE_MS`, 3 min) — the reply then says the page continues below.
7. When the pictures cannot be joined safely (changing header, a fling that skipped content) the segments are saved as numbered parts (`<NN>-<screen>.partK.png`) with the reason.
8. Back to the top: elements of the first screen are remembered; while none is visible the page is more than a scroll away, once they are visible they give the exact distance, so the last scroll lands on the top — no overshoot, no pull-to-refresh, no long press on content that cannot scroll.
9. If something takes over the screen mid-capture or the device refuses a gesture, scrolling stops at once, what was captured is kept, and the reply says what happened and that the phone was left as it is. "Taken over" needs positive evidence (`takeover` in `stitch.ts`), so a screen that gives too little to tell is never stopped:
   - the phone says so: a dialog that was not there, another app in front, the keyboard coming up;
   - otherwise a measured scroll, an element seen once in both reads, or pictures that line up at some scroll distance mean the same screen;
   - only then do vanished elements count: the elements in the header / bottom bar all gone, or none of the old element texts left at all.
   Labels that repeat (the same button on every card) and elements without text are no evidence either way.
10. Elements in page pixels with page refs (`p1`, `p2`…); an element cut by a bar where no other screen shows it whole is marked `partial` (its box is only the visible part).
11. `CAPTURE_TRACE=1` on the API prints every scroll, measured shift and decision (diagnosing a new phone or app).

### Second pass (`services/stitch-v2.ts`)

A step the first pass is unsure about is measured again before the page falls back to parts (`STITCH_V2`: unset or
`rescue` = this default; `off` = the first pass alone, exactly as before; `on` = the second pass for every step).

- **Every shift, to the pixel**: all shifts are scored, the best few re-checked on every row; a scroll by an odd number of
  pixels is no longer missed.
- **Status bar is known chrome**: its height (the same value used for cropping) is given to the joiner, so a clock or
  icon that changes between pictures never hides where the app's own header ends.
- **Edges left out**: rows next to the header and bottom bar (5 % of the screen) are not used to match: while a page is
  scrolled they may fade, carry the header's shadow or still be loading. Such a fade is not taken for a changing header.
- **Repeating content** (identical cards): when several line-ups match equally, the one where the finger's travel says
  the page should be wins (learned from the earlier steps of the same capture); otherwise the step stays unsure.
- **Still strict**: a header or bottom bar that changes as a whole, a fling and an unclear match still give parts.
- **Seams cut mid-overlap**: each seam is cut in the middle of the overlap, on the row where both pictures agree best,
  and checked at full resolution (mean difference ≤ 4 near the cut, ≥ 85 % of the overlap identical) — also on pages
  the first pass joins; when a check fails there, the first pass's join is used unchanged.
- **Scroll container**: on real phones the snapshot carries agent-device's "more content above / below" for scroll
  containers (`DeviceSnapshot.scrollers`); a page that ends exactly at the segment cap is then not reported as
  continuing below. With `STITCH_V2=on` it also ends the scrolling one step early and sizes the scroll from the container.
- **With `STITCH_V2=on` only**: a doubtful seam splits the page there and the rest is still joined (fewer, longer
  parts). Not the default, because Review pairs pages saved in parts part by part.
- **Diagnostics**: every capture's `stitch.seams` records each step (shift, sure or not, which pass, what decided it,
  the next-best line-up, where the seam was cut and how well it matched), `stitch.engine` and `stitch.statusBarPx`.
  When something takes over the screen mid-capture, that screen is kept as a run file (`stitch.stopFrame`).
- **Retake**: a step still unsure (or one that does not match at full resolution at its cut) is taken again, at most
  3 times per capture: first the same screen once it has settled again (a toast or a row still loading may be gone),
  then half a step back (twice the overlap; still below the previous screen, so never at the top). Not for a header or
  bottom bar that changes as a whole — a second picture cannot change that.
- **Repeating content**: when several line-ups tie on the sampled rows, the full-resolution check (every column,
  judged per column block, tolerant of anti-aliasing a pixel up or down) decides when exactly one is identical at its cut.
- **Replay without the phone**: `npm run stitch:replay -w @teststudios/api -- <capture id>` joins a page saved in parts
  again with each pass and prints every step; `-- --all-parts` does it for every page saved in parts; `-- --dir <folder>`
  for any screens taken top to bottom. Real pictures can be checked with `STITCH_REAL_DIR` and
  `apps/api/test/stitch-real.test.ts` (kept outside the repository).

### Joins settled by looking (`services/stitch-choose.ts`, `agent/capture-joins.ts`)

What the measurement still cannot settle is looked at before a page is left in parts:

1. For each doubtful join, up to three candidate line-ups are drawn as strips (the two screenshots joined at that
   line-up, cropped around the cut, red ticks at the cut), lettered A, B, C.
2. **The model looks first** (the project's step model, one call per join): it answers a letter or "none". Its pick is
   used only when the two pictures also agree at that cut.
3. **Then the person in the chat**: a question card with the strips and A / B / C / Keep as parts. They have
   10 minutes (`CAPTURE_JOIN_WAIT_MS`); without an answer the page stays in parts. An unanswered question never marks
   the run as blocked. Schedules, CI, MCP and background jobs never ask.
4. Whatever stays unsure is kept in parts with `stitch.needsJoinCheck`; `stitch.joinCheck` records who settled the
   joins (`ai`, `person`, `none`) and each step's `by` says `ai` / `person` for a chosen join.
5. **Later, without the phone**: Artifacts → Screenshots → **Fix join** on part 1 (pick a strip per join, Join), or the
   agent's `join_capture` (shows the strips; joins at `picks`; `ask_user=true` asks the user). The page is saved as one
   picture in place of the parts — same path and link. Routes: `GET` / `POST /api/projects/:projectId/captures/:captureId/joins`.
   Parts saved before the joiner could measure them that now join cleanly as they are need no pick: Fix join says so
   and joins them, `join_capture` joins them at once (`joinCheck.by: 'auto'`).
6. **What the agent is told**: a page left in parts reaches the agent as one picture of every part side by side,
   numbered in order (not only part 1), and the reply lists each join in doubt — "between part 2 and part 3: <the
   joiner's reason>" — for the agent to quote instead of guessing a cause.

`CAPTURE_JOIN_CHECK=off` turns steps 2–3 off (pages stay in parts as the joiner left them).

## Comparison (`services/screen-compare.ts`)

- Only the app's own elements are compared (other apps' overlays are dropped by resource package); a resource name posing as text (an unlabelled icon) is not text.
- Pairing: resource-id / accessibility id → identical text that appears once on each side (across languages: the same, untranslated element) → role + reading order → nearest position (always marked uncertain; size checks use it only when both sit in the same slot).
- Localization candidates: ellipsis or edge-touching text, extra wrapping, new overlaps, controls missing on one side, untranslated text (same words in both languages, or Latin words on a non-Latin-script screen), moved and reordered elements (top-to-bottom and within a row; right-to-left languages are allowed to mirror).
- A finding on a container and on the label inside it is reported once (on the label); codes and acronyms in capitals are leads (uncertain), not sure findings; partial elements are not size-compared.
- Allowed untranslated: numbers, amounts, ids, links, emails, the project's do-not-translate list (`localization.dnt`), termbase DNT terms and preserve patterns, `project.captureTerms` and the call's `allow_terms`. `remember_terms` adds to `localization.dnt` when the project has Localization settings, otherwise to `captureTerms` — it never creates Localization settings.
- Visual: changed text, missing / moved controls, % of differing pixels against a threshold, a diff image.
- Every reply ends with a fixed reminder: problems only visible in pixels (an ellipsis drawn by the view, clipped glyphs, missing-font boxes, overflow) need the picture checked, even with zero candidates. Candidates are leads, not defects.

## Defect evidence

`file_issue(evidence: [{ capture, marks }])` → for each screenshot: the annotated picture (marks numbered 1…N across the defect), the original, a close-up per mark. Pictures go to the run-file storage the Defects page already reads (`/artifacts/defect-<issue>/…`); annotated pictures and close-ups are also saved under the set's `defects/` folder for downloads. Without `evidence`: a step picture is attached only when that step belongs to the current message (the reply names it); otherwise nothing is attached, the reply says "No screenshot attached" and the chat's latest run is linked for traceability (de-duplication is unchanged).

## Into Localization Review (`loc/capture-review.ts`)

Screens are captured in passes — one language at a time, in any order, across chats and days — and paired at the end.

- **Matching** (`matchCaptures`, read-only): every screenshot set of the project (or the chosen sets / since a time),
  the newest capture of each screen + language. One row per screen in the original language (the project source, or
  `source_language`); per other language a cell is **exact** (same screen name), **suggested** (another name with the
  same layout — element pairing by id, identical text, role + order, plus the page shape; one-to-one, best first,
  never attached without a person's yes) or **missing**. Screens with no original are **extra**. A pair a person
  accepted earlier stays paired, also after that screen is captured again. Languages resolve against the project's
  target locales (`en` ↔ `en-US`); one that is not a target, or that matches two, is shown with the reason and not
  attached.
- **Attaching** (`attachCaptures`): the original-language screenshot becomes an image asset per screen
  (`origin.type: 'capture'`, `<NN>-<screen>.png`, or `… (screenshot).png` when another asset has that name); each
  chosen language's screenshot becomes that language's imported translation (`targets[locale]`, with its `captureId`),
  then `loc_import` runs. A page saved in parts pairs part by part. One long picture is never paired with the same
  page in parts (the whole page would meet its first screen): that language is skipped with a note to join the parts
  first.
- **Strings**: the original's segments are the capture's element texts (boxes in percent, as for read text boxes; a
  container that only repeats a child's text is left out) and the import pairs the two element maps — no picture
  reading. A capture without a map falls back to the picture path (OCR) unchanged.
- **Each picture keeps its own shape and outlines**: the import stores the translation picture's size and where each
  lined-up string sits on it (`targets[locale].layout`, also for pictures read by OCR); Review draws the translation
  at its own size with those boxes, never the original's boxes moved over. A screenshot pair lined up before this is
  rebuilt from the two element maps the first time Review lists it. A page much shorter than the other side, or cut
  short while capturing, is called out.
- **Any route, same pairing**: a saved screenshot passed to `loc_stage_target` onto a screenshot original goes through
  `attachCaptures` (lined up by key), and `loc_ingest_artifact` on one points to `loc_review_from_captures`.
- **Re-running** changes only what changed: an unchanged pair is left alone; a recaptured translation replaces its
  side; a recaptured original becomes the next version, keeps every language it had and the old version is archived
  (with its review history), so each screen shows once in Review.
- **Where people do it**: Review → Translations → New review → **From screenshots** (a table with thumbnails: ✓ same
  name / paired before / in Review, "Looks like … — Use it" for suggestions, any cell can be pointed at another
  screenshot of that language, then **Attach N pairs**); a screenshot review set's **From screenshots** button; and
  Artifacts → Screenshots → **Send to Review** on a chat's set (Localization projects). The agent uses
  `loc_review_from_captures` (`preview`, then `confirm` with the accepted suggestions).
- **Routes**: `GET /api/projects/:projectId/loc/capture-matches?sets=&since=&source=&languages=&screens=&group=` (the
  table) and `POST /api/projects/:projectId/loc/review-sets/from-captures` (`{ cells: [{ sourceCaptureId,
  targetCaptureId, locale? }] }`, attaches exactly those).

## Where files live and how to download them

```
artifacts/screenshots/<set>/            set = chat title (or flow name) + short id
  <language>/<NN>-<screen>.png          the screenshots (…partK.png for a page in parts)
  annotated/<NN>-<screen>-<language>.png
  defects/<issue>-<NN>-<screen>-<language>.png, …-markN.png
  compare/<NN>-<screen>-<a>-vs-<b>.png, …-diff.png
```

- Chat: a card per screenshot (thumbnail, path, size, download), not a workspace tab.
- Artifacts → Screenshots: one group per set, sorted by language then number, with **Download all (.zip)** (`GET /api/projects/:projectId/captures/zip?set=<setId>`): the same layout plus `captures.json`.
- `export_captures`: the same zip as a file in the chat.

## Agent guidance

The workflow is in `skills/mobile-screenshots/SKILL.md` (routed from `qa-lead`, `mobile-qa-lead` and `loc-lead`; `mobile-localization` and `mobile-bug-report` point to it) and one line in `agent/prompts.ts`. Screenshots for Localization Review follow `skills/loc-app-screenshots/SKILL.md` (routed from `loc-lead`): pick the surface, one language per pass, the same screen names, coverage after each pass, then pair and attach. It is written as rules, not examples, so the agent applies it to any app, language or comparison.

## Saved flows

A recorded `capture_screen` step is replayable: a cached replay captures again with the same screen, language and options (replacing the file; the run keeps its own copy).

## Tests

`apps/api/test/{captures,stitch,stitch-v2,stitch-choose,stitch-real,annotate,compare,captures-export,capture-review,capture-surfaces}.test.ts`,
`packages/device-driver/test/real-scrollers.test.ts`,
`apps/web/e2e/captures.spec.ts`, `apps/web/e2e/zz-loc-review-screenshots.spec.ts`.
