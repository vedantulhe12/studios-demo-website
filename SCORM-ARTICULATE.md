# Articulate and government SCORM

Research notes on the packages localization has to open, translate, repackage, play, score, and resume.

Compatible means all four of these, on the same zip, for one target locale:

1. Ingest finds the learner-visible text and the voice media.
2. Export rewrites only that text and returns a zip the original player still launches.
3. The course player runs the published runtime and records completion, score, and resume.
4. A walk can open the course, follow the path a learner can actually take, and flag strings that stayed in the source language.

## Interactivity levels

Articulate’s scale is a design rating of what the learner does inside **one** package. It is not a SCORM edition. Source: [Get to Know The 4 Levels of E-Learning](https://community.articulate.com/blog/articles/get-to-know-the-4-levels-of-e-learning/1081372).

| Level | Learner action | Typical build | What grows in the zip |
| --- | --- | --- | --- |
| 1 | Clicks Next | Static slides, simple quiz | Small course-data script, a few images |
| 2 | Clicks, drags, matches | Audio, short video, drag-and-drop | More media, one straight path |
| 3 | Chooses a path | Branched scenario, layered feedback, characters | More slides and variables. Every branch’s text is in the course-data script |
| 4 | Plays | Game, score, timer, simulation, 360° image, avatars | Same single package, often 50–200 MB, plus a long resume blob |
| 5 | Articulate does not define this | Other people mean a full simulation, or the same course published as cmi5 or xAPI | Same Storyline or Rise files, plus `cmi5.xml` or `tincan.xml` |

Gamification sits at level 4. A level 3 course can still be scored. A level 4 course adds points, a timer, object states, and a result that changes the next screen.

Storyline and Rise publish **one SCO** at every level. The manifest launches one player file. Branches, scores, and locks live inside that player. A multi-SCO sequencing manifest (the Rustici golf courses) is a different shape, and Articulate courses rarely use it.

SCORM edition is a second axis, independent of level:

| Publish target | Use it for |
| --- | --- |
| SCORM 1.2 | Short courses. Resume string cap is 4,096 characters |
| SCORM 2004 2nd, 3rd, 4th Edition | Branched and gamified courses. 4th Edition is the one Articulate recommends when resume data is large |
| xAPI | Same course, different wrapper. Debug mode in `scormdriver.js` does not apply |
| cmi5 | The practical step past level 4. Same lesson, `cmi5.xml` added |

A level 4 game published as SCORM 1.2 fails resume because the compressed `suspend_data` string exceeds 4,096 characters. Publish level 3 and level 4 as SCORM 2004 4th Edition or cmi5. [Resume and suspend data](https://community.articulate.com/blog/articles/learning-more-about-your-lms-resume-behavior-and-suspend-data/1135907).

`suspend_data` is sent after each slide. It stores responses, navigation, object states, variables, and interaction results. Storyline compresses it. The LMS stores the string and returns it. Only the course runtime decompresses it. Round-trip the string unchanged.

## Articulate package anatomy

Storyline 360, Presenter 360, Quizmaker 360, and Engage 360 share one LMS driver. Rise 360 uses the same driver in a different folder.

### Storyline (and Presenter, Quizmaker, Engage)

| Path | Role |
| --- | --- |
| `imsmanifest.xml` | One SCO. Launch href points at the player |
| `index_lms.html` | LMS launch file |
| `lms/scormdriver.js` | LMS driver. `strLMSStandard` names the edition (`SCORM`, and the edition beside it). Leave this file byte-identical |
| `html5/data/js/data.js` and sibling slide scripts | `window.globalProvideData('…', '<json>')`. Learner text is in fields such as `lmstext` |
| `story_content/frame.xml` | Player chrome labels |
| `story_content/` video, including `.hls` folders | Media. A missing `.m3u8` means the zip was repacked and dropped folders |
| `mobile/` | Duplicate rasters. A 30 MB `.story` can publish as a ~200 MB zip because of these PNGs |
| HTML comment | `Created using Storyline 360` plus a player version |

Publish settings live in `data.js` on the `threshold` object: `type` is `view`, `quiz`, or `action`; `passPercent` and `viewThreshold` match that type; `passStatus` and `failStatus` are the reporting pair (passed/incomplete, completed/incomplete, and so on). [How to read them back](https://community.articulate.com/discussions/discuss/review-scorm-publish-settings-after-published/1216239).

Newer Storyline builds split slides into several `html5/data/js/*.js` files. Each one calls `globalProvideData`. Read every such file, not only `data.js`.

### Rise 360

| Path | Role |
| --- | --- |
| `scormcontent/index.html` | Launch content |
| `scormdriver/scormdriver.js` | Same driver, different folder. `SHOW_DEBUG_ON_LAUNCH` lives here |
| Locale bundles under `scormcontent/` | One course, several languages, when the author used Rise localization |

The public Articulate sample of that last shape is a staff-posted Rise course, three languages, SCORM 2004 3rd Edition, tracked as passed/incomplete: [E-Learning Heroes thread](https://community.articulate.com/discussions/discuss/sample-scorm-multilanguage-file-from-locatisation-feature/1256937).

### Files the export must preserve byte-identical

- `lms/scormdriver.js` and `scormdriver/scormdriver.js`
- `html5/lib/**` (player engine)
- Minified vendor libraries, fonts, CSS, schemas (`.xsd`, `.dtd`)
- Compressed `suspend_data` (the player stores it; the filter does not rewrite it)
- Video and audio bytes, except when a media job replaces a localized recording or adds a caption file on purpose

Repack with the archive root the manifest expects. Storyline zips sometimes hide `story_content` subfolders from Windows Explorer; JSZip-style tools see them. A rezip that flattens those folders breaks video.

## What this repo does today

Localization seams, from [LOCALIZATION.md](LOCALIZATION.md):

| Seam | Owns |
| --- | --- |
| `packages/loc-formats` | Detect, extract, merge. No imports from `apps/` |
| `apps/api/src/loc/` | Ingest, segments, player, walk. Calls the filters |
| `playerHtml` in `apps/api/src/loc/preview.ts` | Course player. `scorm-2004` vs 1.2 is one flag. Runtime is scorm-again |
| `apps/api/src/loc/scorm-walk.ts` | Page list from the manifest, CMI trace |

SCORM filter: `packages/loc-formats/src/filters/scorm.ts`.

- Versions it names: `scorm-1.1` (`CSF.xml`), `scorm-1.2`, `scorm-2004`, `unknown`.
- 2004 2nd, 3rd, and 4th Edition all become `scorm-2004`. Sequencing XML is copied through on export.
- Quiz strings it understands are Rustici-style `questions.js` (`new Array` of choices) and `contentfunctions.js`.

Classifier: `packages/loc-formats/src/classify.ts`.

- `questions.js` → `scorm-questions`. `contentfunctions.js` → `scorm-ui`.
- A script whose name matches the SCORM runtime list (`scormdriver`, `tincan`, `xapiwrapper`, …) is skipped.
- A script under `lib/`, `vendor/`, or similar is skipped.
- A script that looks minified (long lines, or a license header) is skipped.
- HTML goes to the HTML filter. `vtt` / `srt` go to subtitles. Audio and video become media jobs.

Zip safety: `packages/loc-formats/src/zip.ts`. Default caps are 20,000 entries, 2 GB uncompressed, compression ratio 400. A 200 MB Storyline publish fits. A zip-bomb or a `..` path does not.

Fixtures, from [fixtures/loc/README.md](../fixtures/loc/README.md):

| File | What it actually is |
| --- | --- |
| `scorm/golf-1.1.zip` | Rustici “Golf Explained” SCORM 1.1. CC BY 3.0 US. HTML pages plus `questions.js` |
| `scorm/mini-1.2.zip`, `scorm/mini-2004.zip` | Hand-built by `node fixtures/loc/make-scorm.mjs`. Same golf shape |
| `scorm/media-rich.zip` | Hand-built 1.2 package with PNG, SVG, video, VTT, DOCX, PDF, and a vendor `APIWrapper.js` that must stay untouched |

Those fixtures prove the golf-shaped reader. They do not contain `globalProvideData`, `lmstext`, `scormdriver.js`, or Rise locale bundles.

## Gap that blocks Articulate

Storyline lesson text lives in `window.globalProvideData` JSON (`lmstext` and the display strings beside it). That file is one dense script, so `looksMinified` classifies it as a library and ingest skips it. The course then plays in English after a “successful” export.

Rise lesson text lives under `scormcontent/`, not in a pile of HTML pages the HTML filter already walks.

The player can still launch a package whose runtime is self-contained, because the iframe loads `index_lms.html` or Rise’s content and the driver talks to scorm-again. Launch without extracted strings is not compatibility.

`cmi5.xml` and `tincan.xml` are not a detected version in `parseImsManifest`. A cmi5 Storyline zip usually still has `imsmanifest.xml` plus the same `html5/data/js` tree. Detect the wrapper, and extract text from the same course-data files.

## Sample corpus

Use real publishes to prove the reader. Keep the golf fixtures as the regression set for manifests, sequencing, and `questions.js`.

Articulate does not ship a catalog of level-labeled SCORM zips. Free **source** templates (open in Storyline 360, then publish) are on [Storyline templates](https://community.articulate.com/category/free-downloads/blog/storyline-templates).

| Course | Level | Where |
| --- | --- | --- |
| Job interview branched scenario | 3 | [Download](https://community.articulate.com/download/storyline-job-interview-branched-scenario) (`.story`; publish it) |
| Gamified quiz with timer, Jeopardy-style game | 4 | Same templates index |
| Crack the Safe | 4 file layout, web publish | [GitHub](https://github.com/VitorDeSenna/Crack-the-safe-The-Game). `lmsPresent` is false, so it tests `html5/` and `story_content/` and does not test LMS scoring until republished |
| Time Management | Published Storyline SCORM, lighter than a game | [Time.Management.zip](https://github.com/NeilFenech/SCORM-Time-Management/releases/download/SCORM/Time.Management.zip) (~4 MB) |
| Rise, three languages | Rise localization output | [Heroes thread](https://community.articulate.com/discussions/discuss/sample-scorm-multilanguage-file-from-locatisation-feature/1256937) |

Three publishes cover the ladder: the interview scenario as SCORM 2004 4th Edition, the timed game as SCORM 2004 4th Edition, and that same game as cmi5.

Sequencing samples (golf, not Articulate), from [Rustici’s golf examples](https://scorm.com/scorm-explained/technical-scorm/golf-examples/):

- [AllGolfExamples.zip](https://scorm.com/wp-content/assets/golf_examples/PIFS/AllGolfExamples.zip)
- [Single SCO, 2004 3rd](https://scorm.com/wp-content/assets/golf_examples/PIFS/ContentPackagingSingleSCO_SCORM20043rdEdition.zip)
- [Forced sequential, 2004 3rd](https://scorm.com/wp-content/assets/golf_examples/PIFS/SequencingForcedSequential_SCORM20043rdEdition.zip)
- [Post-test rollup, 2004 4th](https://scorm.com/wp-content/assets/golf_examples/PIFS/SequencingPostTestRollup4thEd_SCORM20044thEdition.zip)
- [Basic runtime calls, 2004 3rd](https://scorm.com/wp-content/assets/golf_examples/PIFS/RuntimeBasicCalls_SCORM20043rdEdition.zip)

`golf-1.1.zip` in this repo is already the 1.1 package from that set.

## Government packages

The local Chrome downloads are a second corpus. They are complex, they include voice activities, and they come from US government and Indian government programs. Their authoring tool is unknown until the zip is opened. Treat “government” as the customer, and the fingerprint as the format.

Public patterns, which describe the field and not these specific files:

- US federal courses are often SCORM 1.2 or SCORM 2004. Authoring is commonly Adobe Captivate, Storyline, or bespoke HTML. One public USDS curriculum (DITAP) is LiaScript exported to SCORM 1.2 and 2004; that fact does not identify the local downloads.
- Indian procurement language (BIS NITS LMS and e-learning RFP, July 2024) asks for SCORM 2004, content built in Adobe Captivate or Articulate or a current equivalent, Hindi and English with a learner-facing language choice, human voice in an Indian accent, and screen-reader accessibility. [RFP PDF](https://www.bis.gov.in/wp-content/uploads/2024/07/NITS_-LMS_18072024.pdf).
- iGOT Karmayogi (Department of Personnel & Training) is the civil-service learning platform. CDAC MeghSikshak accepts a SCORM zip alongside HTML, PDF, and MP4.

### Voice activities

Voice shows up in three layers. Handle each layer with the pipeline that already owns it.

| Layer | What you see in the zip | What to do |
| --- | --- | --- |
| Narration | `mp3`, `wav`, `m4a`, sometimes a matching `vtt` / `srt` | Media job: transcribe, subtitle, and let a localized recording replace the audio when the product already does that |
| On-screen transcript | The same words in HTML, `lmstext`, or a caption cue | Text segment, linked to the cue when both exist |
| Speak-and-compare | Custom script, recorder, Web Speech, or a web object that scores an utterance | Fingerprint the script. Extract visible prompts. Leave the scoring code byte-identical. The utterance itself is not a translatable string |

A Hindi/English switch inside one Indian government package is learner-facing bilingual content. Extract both languages as source text with a locale tag. Producing a new locale zip (for example German) is a separate export. Collapsing the switch into a single language drops a path the learner can still open.

### Fingerprints

Run this before writing a reader. Check paths and the first 2 KB of the launch HTML and the largest `.js` file.

| If you find | Authoring tool |
| --- | --- |
| `Created using Storyline 360`, `html5/data/js/`, `globalProvideData`, `lms/scormdriver.js`, `story_content/` | Storyline family |
| `scormcontent/index.html` and `scormdriver/scormdriver.js` | Rise 360 |
| `project.txt` mentioning Adobe Captivate, `CPM.js`, Captivate runtime folders (`ar/`, `dr/`, assets next to a Captivate `index.html`) | Adobe Captivate |
| `questions.js` plus `contentfunctions.js` and golf-style HTML | Rustici sample shape (already supported) |
| LiaScript markers, or markdown-derived HTML with a thin SCORM wrapper | LiaScript / custom export |
| `imsmanifest.xml` and ordinary HTML, none of the markers above | Bespoke HTML course. Use the HTML filter and the audio/subtitle path |
| `cmi5.xml` or `tincan.xml` beside any of the above | Wrapper only. Identify the course files with the other rows |

Captivate is the likely neighbor of Articulate in government bids. Its text is not in `lmstext`. A Storyline reader applied to a Captivate zip will corrupt the package.

Variable names, ids, URLs, `cmi.*` keys, and LMS status words stay out of segments. `isNaturalLanguage` in `classify.ts` is the existing rule for that.