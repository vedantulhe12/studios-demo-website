# Phase 1 review: Video Translate rebuild

Reviewer: independent agent that did not write the code. Date: 2026-10-07. Branch `video-rebuild` at `e3ce2d2` (5 commits ahead of `dev` by `git log --oneline dev..HEAD`; the brief said 6). No product code or tests were changed. `.env` was not read and the Node API was not started.

## Verdict

**The Phase 1 gate is not met.** No job has ever run with the real model, so "upload, analyse, translate, export with a live trace of real work" has not happened once; the agent loop, the trace and the web screens have only been driven by hand-written scripts and a fake worker. Independently of that, the deterministic pipeline fails three gate rows on its own evidence: a connector ring is half erased on clip-G1 (F1.3), two of five perturbed-G1 variants fail (rule 2.5's proof), and the layout row fails for two clip-G1 blocks. Remnants on clip-G1 are not measured by any test. `docs/STATUS.md` says most of this plainly and its numbers reproduced when I reran them. What it does not say is listed in the findings below: one trace row will print a wrong number in a real run, the product's own QA cannot see the damage the test suite sees, a job can finish as "passed" without QA having run, and two tests pass for reasons weaker than their names.

## What I ran

| Command | Result | Matches STATUS.md |
| --- | --- | --- |
| `uv run pytest -q -p no:cacheprovider harness/tests tests/test_g15_onscreen.py tests/test_tools.py` | 32 passed in 191 s | yes |
| `OBJECT_STORE=local node --import tsx --test apps/api/test/vt-*.test.ts` | 15 passed, 0 failed | yes (15) |
| `node scripts/check-no-handoff-refs.mjs` | "735 product files scanned, none references handoff/" | yes |
| `node scripts/check-traceability.mjs` | "122 stories, 328 criteria, 0 PASS claim(s)" | yes |
| `uv run pytest … tests/test_g1_onscreen.py` (warm segment cache) | 1 failed, 11 passed in 116 s | yes |
| same, cold cache (`VW_SCRATCH=/tmp/…`) | 1 failed, 11 passed in 249 s | yes |

On both clip-G1 runs the only failure was `test_F1_3_no_numeral_or_neighbouring_object_is_partly_erased` with `[('B1962', 'numeral', 86)]`, exactly as STATUS.md states. The cold run took 4 min 9 s for decode, analysis, 16 block models, render, encode and the measurements, consistent with the "about 3 minutes" claim for the pipeline alone.

Not rerun: `tests/test_g1_perturbed.py`. The existing log `services/video-worker/.scratch/perturbed.log` shows "3 failed, 23 passed" with the rescale and re-encode failures STATUS.md describes. The 14 Playwright specs were not run.

## Findings, most severe first

| # | Severity | What | Evidence | Suggested fix |
| --- | --- | --- | --- | --- |
| 1 | Blocker | No real-model run. The gate needs one; the brief's release rule needs one. The request shape in `buildRequest` is consistent with the current API reference but has never been sent. | `docs/STATUS.md:11`; `apps/api/src/video/agent/loop.ts:71-87`; `apps/api/test/helpers-vt.ts:116-128` | Add the key, run clip-G1 and clip-G15 end to end, record the run, review again. |
| 2 | Blocker | F1.3 fails on clip-G1. The left half of the connector ring beside the 1962 caption is erased on every frame from the block's arrival to the end of its page. It is obvious at normal size, not a pixel-peeping defect. | `docs/reviews/phase-1/zoom_1962_ring_f647_f660_f846.png`, `settled_2.png`; test output above | The ring is ink-coloured and letter-sized and falls inside a line box; the by-size rule in `vw/layers.py:310-319` does not catch it here. Needs a real fix, then a test on more than the numeral box. |
| 3 | Blocker | Rule 2.5 is not proven. Rescale: one block's first visible frame is one frame late. Re-encode: one caption's ink box grows 8 px into a connector ring. | `.scratch/perturbed.log` (not rerun) | As STATUS.md says. Until both pass, "no G1-specific tuning" is unproven for thresholds in `layers.py` and `structure.py`. |
| 4 | High | The product's own `qa_suite` cannot see finding 2. It has no check for damage to protected objects, no onset check and no motion check, yet the tool description given to the model says it measures "onsets, motion". On clip-G1 a real job's QA report would not mention the ring. | `vw/tools.py:448-484`, `vw/qa.py`; `apps/api/src/video/agent/tools.ts:56` | Port the harness's F1.3, H1 and H2 measurements into `qa_suite`, or correct the description and mark those rows as not measured in the QA report. |
| 5 | High | A trace row will show a wrong number in a real run. `render_preview` returns `changed_pixels` and `left_untouched`; the API reads `blocks_drawn` and prints "0 blocks drawn on frame N". The fake worker returns `blocks_drawn: 1`, so the tests cannot see the drift. | `vw/tools.py:426`; `apps/api/src/video/agent/tools.ts:227`; `apps/api/test/helpers-vt.ts:69`; `docs/VIDEO-API.md:55` | Return the real count from the worker. Add one contract test that runs the API's `describeResult` over real worker output for every tool. |
| 6 | High | A job is marked `passed` whenever the model calls `finish` with an empty `unresolved` list. Nothing requires an export, a QA run, or a passing QA. | `apps/api/src/jobs/video-worker.ts:66-70`; `apps/api/src/video/agent/loop.ts:362-366` | Derive the job status from stored facts: export present, QA present, QA passed, guard skips. Treat the model's summary as commentary. |
| 7 | High | Only the first audio track survives export. The second `inp.demux(s)` runs on a container already at end of file and yields nothing. I confirmed it on a two-track file: source 45 and 45 packets, output 45 and 0. `qa_suite` does catch it (`audio1` fails), but `probe` tells the user "2 audio tracks; all are kept". | `vw/render/encode.py:191-195`; `vw/tools.py:93`; reproduction in "What I ran" notes below | Demux all audio streams in one pass. Add a multi-track clip-G15 variant. |
| 8 | Medium | `test_F3_1_no_pixel_changes_outside_the_declared_mask` proves nothing on a warm cache. `changed_outside_mask` is the literal `0`; the real enforcement is the raise in `render_frame`, which only runs for segments that are re-encoded. With cached segments no frame is rendered, `max_changed_share` is 0.0 and the test passes. My cold run did exercise it and passed. | `vw/pipeline.py:345-347, 389-390`; `vw/render/encode.py:147-151`; `tests/test_g1_onscreen.py:44-47` | Have the test clear the segment cache (as `test_tools.py` does), count checked frames in the report and assert the count equals the frame count. Report a measured value, not a constant. |
| 9 | Medium | The declared mask is whatever the block model calls text. It is tight (old ink or new ink, grown one pixel, then rounded to chroma samples; `vw/render/compose.py:94, 110-113`), and the test bounds it at 8 % of the frame. But "0 pixels outside the mask" passes while the ring in finding 2 is erased, because the ring is inside the mask. F3 is a necessary check, not evidence that only text changed. | `vw/render/compose.py:94-117`; `tests/test_g1_onscreen.py:47` | Add a test that the mask does not intersect anything that predates the text or extends beyond the line boxes, independent of the model's own `tau`. |
| 10 | Medium | Lowered threshold in a passing test. F2.2's limit is 3 grey levels at the 99th percentile; the test asserts 4.0 and is green. The map marks F2.2 FAIL and STATUS.md discloses 3.8, so the documents are honest, but rule 2.4 says not to lower thresholds, and F1.3 was handled the right way (left red). | `tests/test_g15_onscreen.py:101` | Assert 3.0 and let it fail, or split it into its own red test. |
| 11 | Medium | `test_tools.py` ignores the layout check when asserting the QA suite. The synthetic clip's own Hindi wording fails layout and the test filters it out. | `tests/test_tools.py:54` | Use wording that fits, or assert the failure explicitly with its numbers. |
| 12 | Medium | Remnants (F1.1, F1.2) are not measured on clip-G1 by any test. The harness OCR scanner the brief asks for does not exist; the only read-back is the product's own OCR, at one frame per block, on clip-G15. By eye I found no leftover English on clip-G1 (see below), but nothing measures it. | `tests/test_g1_onscreen.py` (no such test); `vw/qa.py:68-89` | Build the independent scanner; run it on every frame class (settled, reveal, pan). |
| 13 | Medium | Typography on clip-G1 block B2022: the source heading is bold and the bullets light; the output draws all lines in one light weight. Square bullets became round ones and the hanging indent of wrapped bullet lines is lost. In several other blocks the Hindi reads lighter than the source (B1949, B1986). One weight is measured per block. | `docs/reviews/phase-1/settled_4.png`, `settled_1.png`, `settled_2.png`; `vw/pipeline.py:228` | Measure weight and indent per line; carry list markers through unchanged. |
| 14 | Medium | Renders are not byte-reproducible. A cold render of clip-G1 produced different video packets from the cached render of the same inputs (7,009,268 against 7,009,326 bytes; audio identical). I did not find the cause. Separately, `spec_digest` hashes the text coverage and schedule but not the object layer, ink colour, panel fit or background, and relies on a hand-bumped `PIPELINE_VERSION`, so a code or model change can silently reuse stale segments. | packet hashes `0b548fb90736` against `b4cfa807ebb0`; `vw/pipeline.py:350-372` | Find the source of the difference (encoder threads or analysis). Hash every input the compositor reads, plus a hash of the pipeline source. |
| 15 | Medium | Only Hindi and Latin can be rendered. Any other script raises `KeyError` in `typo.shape`, which `tools.call` reports as "Missing parameter 'Arab'". The tool schema offers `"Arab"` as an example and the API accepts any language string. Text is always left-aligned at the source's left edge; nothing is right-to-left. | `vw/typo.py:26-27, 99-100`; `vw/tools.py:506-507`; `apps/api/src/video/agent/tools.ts:52` | Reject unsupported target languages at `POST …/translate` with a plain message, and return a proper tool error. |
| 16 | Medium | Export breaks outside clip-G1's shape. Always muxes MP4, including to a file named `.mov`, and for `.mkv`/`.webm` sources whose audio codecs MP4 may not carry. Rotation is copied only from a `rotate` metadata key, which current FFmpeg exposes as display-matrix side data. Subtitle and data streams are dropped. Colour tags are set inside `try/except: pass`. 10-bit and HDR are refused. The whole video is decoded to raw planes on disk (about 2.9 GB for clip-G1; a 60-minute 1080p file would need hundreds of GB) and all encoded packets are held in memory before muxing. | `vw/render/encode.py:50-54, 138, 176-178, 141-158`; `vw/tools.py:434-435`; `vw/source.py:22-26, 95-106` | Out of Phase 1 scope except where STATUS.md implies otherwise; list these under known gaps. |
| 17 | Medium | Design assumptions that fit clip-G1's structure. No literal clip values, but: camera motion is translation only; the camera-path plot draws x only and is labelled "travel left to right"; text is read on one settled frame per quiet stretch, so text visible only during motion is never found; a block is one rigid layer with one opacity and one offset; the default script everywhere is `"Deva"`; a comment and rule in `layers.py` describe "a connector line ending in a ring beside a caption". | `vw/analysis/structure.py:219-276`; `vw/tools.py:270-280`; `vw/text/detect.py:94-102`; `vw/pipeline.py:50, 217, 225`; `vw/layers.py:310-319` | Acceptable for Phase 1 if stated. The perturbed and clip-G15 tests are what stop these becoming tuning; two of them fail. |
| 18 | Low | The tool description for `crop_zoom` gives `"the 2024 caption"` as its example title. It is the brief's own example and is also a caption in clip-G1. The system prompt itself has no clip knowledge. | `apps/api/src/video/agent/tools.ts:29`; `apps/api/src/video/agent/prompts.ts:7-27` | Use a neutral example. |
| 19 | Low | "RECORDED-REPLAY" is a misnomer. The model messages are hand-written scripts and the worker results are invented; nothing was recorded from a real run. Events do carry `replay: true`, but the web trace does not show that flag anywhere. | `apps/api/test/helpers-vt.ts:1-6, 116-128`; `apps/web/src/video/trace/reducer.ts:29` (stored, never rendered) | Call them scripted. Record a real run once a key exists. Show a "replay" label in the UI. |
| 20 | Low | Stage bar is honest but thin. No ETA, `done` or `total` is ever published (`etaFromPace` is never called), and no progress rows are emitted during long worker calls, so the "one real row every 15 s" rule will not hold during a multi-minute export. `rebuild` is marked running whenever `remove` is; `animate` is driven only by `render_preview`. Nothing is faked. | `apps/api/src/video/stages.ts:18-19, 69`; `apps/api/src/video/agent/tools.ts:54` | Stream worker progress (the worker already has `on_progress`/`on_segment` hooks that the tool layer ignores). |
| 21 | Low | Security and privacy, quick pass. Organization scoping is applied on every route and the `personal:<userId>` workspace cannot be named by another user (`routes/video.ts:35-42`); artifacts need a per-organization record and a 64-hex hash (`artifacts.ts:49-50`); upload paths are server-generated and names sanitised. Gaps: (a) unhandled errors return `err.message` to the client, which can carry server paths (`routes/video.ts:26`); (b) the worker has no authentication, accepts any absolute path at `POST /sources`, and serves any registered export; it is safe only because it binds to 127.0.0.1 (`vw/server.py`, `run.sh`); (c) the worker's source cache and image store are shared across organizations by content hash, and `origin.json` is overwritten by the latest uploader of the same bytes (`vw/tools.py:53-54`); (d) the export route downloads whatever path the worker reported, without checking it is under the data directory (`routes/video.ts:214-215`); (e) `answerQuestion` does not check the option against the question (`service.ts:185-190`); (f) any member of an organization can read every video in it, with no project-level check, and a member without `loc:translate` can still upload and translate in their personal workspace (`routes/video.ts:44`); (g) abandoned uploads are never cleaned up and there is no quota. | as cited | (a) and (b) before any shared deployment; the rest can be scheduled. |
| 22 | Low | Dead assignment: `k` is computed then overwritten in `t_layout_fit_check`. | `vw/tools.py:396-397` | Remove one. |
| 23 | Low | `docs/demos/phase-1/` holds five screenshots, not the Playwright screen recording the brief asks for, and they come from the mocked API. | `ls docs/demos/phase-1` | Record once a real run exists. |

Reproduction for finding 7: a 30-frame 64 x 64 H.264 file with two AAC tracks, exported through `vw.render.encode.export` with a pass-through renderer; `qa.container_checks` then reports `audio0 True 45 of 45`, `audio1 False 0 of 45`.

## What I saw in the output

Source against the product render `services/video-worker/.scratch/g1_product_hi.mp4` as it stood before my runs (the clip-G1 test rewrites that file each time it runs, so it now holds my cold render; see finding 14): all 16 blocks at rest, six blocks mid-reveal, four pan frames and the last frame. Images are in `docs/reviews/phase-1/`.

- **Leftover source text:** none visible at rest, mid-reveal or during pans. Difference maps show changes only where captions are, plus faint encoder noise.
- **Numerals:** all 16 intact, including where the source text ran under them (1945, 1949, 1956, 1961, 2022).
- **Connectors and rings:** one damaged, the 1962 ring (finding 2). I looked for others and found none; the rings beside 1945, 1949, 1956, 1961, 1980, 1986, 1991, 1996, 2002, 2012, 2018, 2020, 2021, 2022 and 2024 are whole.
- **Placement:** Hindi sits inside its panel in every block. B2012's last line ends close to the panel's bottom edge and B2020's last line runs close to the right edge; neither is visibly clipped. Line spacing is wider than the source's by design.
- **Reveals:** on the sampled frames the Hindi has the same partial opacity and vertical offset as the English.
- **Pans:** the Hindi travels with the content; no smearing or patches.
- **Patches and seams:** none visible on the grey panels or dot grid, apart from a slightly brighter area where the 1962 ring was removed.
- **Weight and style:** finding 13.
- **Scan for unexpected changes:** at five settled frames, pixels differing by more than 40 levels outside the captions' boxes are either captions carried over from an earlier page or isolated specks of 6 to 15 pixels at a caption's lower edge.

The wording in this render is the hand-made reference's, supplied by the test. Nothing here says anything about what the agent would write.

## Traceability

No criterion is marked PASS, so `check-traceability.mjs`'s main check (PASS rows must name a real test) checks nothing at present. All 26 PARTIAL rows name tests that exist. Of the rows I read against their tests:

| Criterion | Assessment |
| --- | --- |
| H2.2 (sub-pixel placement, no shimmer) | Overclaims. `test_H2_camera_travel_measured_to_a_tenth_of_a_pixel` checks the pipeline's internal track against the generator, not where text lands in the output. |
| H2.3 (measured tracks beat a fitted model) | Overclaims. Same test; no fitted model exists to compare with. |
| H1.1, H1.2, H2.1, H7.2 | The note says "passes on clip-G1 and clip-G15". Only clip-G1 has a black-box test for these. The clip-G15 test compares the pipeline's own `plan.states` with the generator, which is what H1.3 says not to do. |
| L1.2, M4.1 | The 45 dB figure is the product's own number, using its own mask, and is read from the segment cache on a warm run. It is not measured by the harness. |
| F3.1 | See finding 8. |
| F2.1 | Honest; note the per-frame gain and offset fit before comparing, which absorbs any global level shift. |
| D1.2 | Mild overclaim: last visible frame and per-frame geometry are not tested; fully-visible is allowed to be one frame off on clip-G1. |
| F1.2 | Honest about using the product's own OCR; one frame per block only. |
| A3.3, L1.1, K4.1, W7.2, D1.3, H3.1 | Honest. D1.3's note that 5 extra groups are proposed and left for the agent is the kind of detail STATUS.md's "16 of 16" omits. |

## Verified as true

- Every count and failure in STATUS.md that I reran reproduced.
- No file under `services/video-worker/vw`, `apps/api/src/video`, `apps/api/src/routes/video.ts`, `apps/api/src/jobs/video-worker.ts`, `apps/web/src/video` or `packages/shared/src/video.ts` contains a clip-G1 year, name, dimension, frame number, coordinate, block count or Hindi string, or a reference to `handoff/`. Thresholds are expressed relative to line height, noise level or frame size.
- The oracle is used only by tests, and only for wording, matching by English text and measurement regions.
- Rules 2.2 and 2.3 on clip-G1: timestamps come from the source list, the time base and sample aspect ratio are copied, audio packets are copied without decoding and the mux has no shortest-stream option (`vw/render/encode.py`). The harness confirms identical timestamps and 1,587 identical audio packets on a cold run.
- Without a key a job fails with a plain message before any trace is opened (`apps/api/src/jobs/video-worker.ts:51-54`); there is no stand-in model in product code.
- Features that do not exist are disabled and labelled in the UI, and `voiceover`/`lipSync` are refused with 422.
- Trace rows are built from tool results and model output only; images shown to the model are fetched by hash and the hash is checked (`apps/api/src/video/artifacts.ts:57`).

## Not checked

- Anything involving the real model, and therefore the trace's real content, pacing, cost and time.
- `tests/test_g1_perturbed.py` (read from the existing log only) and the Playwright specs.
- The web player's frame accuracy, and the web screens beyond a read for invented progress.
- Bitstream sample aspect ratio and rotation (STATUS.md also lists these as unchecked).
- Variable-frame-rate, B-frame, non-MP4 and 10-bit sources through the product: read from the code only, except the two-audio-track case.
- Licences in `docs/MODELS.md`.
- Translation quality: the wording under test is the reference's, not the product's.
