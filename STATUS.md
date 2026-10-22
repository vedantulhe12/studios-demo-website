# Status: Video Translate

Where the rebuild stands. The per-criterion detail for all 122 stories is in [`TRACEABILITY.md`](./TRACEABILITY.md); the plan is in [`PLAN.md`](./PLAN.md); what exists and what will be removed is in [`AUDIT.md`](./AUDIT.md); licences are in [`MODELS.md`](./MODELS.md); the contracts between worker, API and web are in [`VIDEO-API.md`](./VIDEO-API.md).

Last updated: 2026-10-07 · branch `video-rebuild` · nothing pushed or deployed · no production database or bucket touched.

## The short version

- The on-screen translation pipeline works from the video alone and passes its black-box gates on clip-G1, with the wording supplied by the test. It beats the hand-made reference on the one motion criterion the reference misses.
- The API, the Opus 5.5 agent loop, the live trace and the four web screens are built and tested against recorded replays and mocks.
- **Real runs have now happened** (2026-10-08, local stack, Opus 5.5 through the Claude API). A small synthetic clip: completed in about 2 minutes, 7 of 7 checks. **clip-G1: completed in 16 minutes** (the target is 10), 523 trace events including 138 images and 56 thinking summaries, 16 captions translated, logos left alone by the agent's own classification, timestamps and all 1,587 audio packets identical, 0 pixels changed outside the mask. **6 of 8 of its own checks passed; it finished with 6 things not fixed** (listed below). Frames from that export are in `docs/demos/phase-1/real-run/`.
- Voice, lip-sync, the editing controls and any-video breadth (Phases 2 to 5) are not built.

## Phases

| Phase | Scope | State |
| --- | --- | --- |
| **0** | Audit, plan, licences, traceability, harness, synthetic and perturbed clips | **Done**, except the voiceover half of the gate (see below) and an independent OCR in the harness |
| **1** | Upload → analysis → on-screen translation → export, with the live trace | **Runs end to end with the real model; gate not closed.** clip-G1 completes with open defects and over the time target |
| 2 | Review and editor, select-and-tell, partial re-render | Player, compare modes, timeline and read-only tabs exist. No editing, no select-and-tell. Partial re-render is proven in the worker |
| 3 | Voice, speakers, lip-sync | Not started |
| 4 | Any-video breadth | Not started. Known failures on colour-on-colour text and occluders are listed below |
| 5 | Speed, reliability, privacy, polish; removal of the old pipeline | **Old on-screen studio removed (2026-10-08)**: the studio screen, the old on-screen and mux code, its routes and tests are gone. A video in a localization project now has a "Translate this video" button that copies it into Video Translate; subtitles and audio dubbing in localization are unchanged. Old stored results are removed by `scripts/purge-old-onscreen-data.mjs` (dry run by default). Speed, reliability, privacy and polish: not started |

## What is measured, and on what

All numbers are from tests in this repo, run on this Mac (Apple M2, CPU only).

**Harness on the hand-made clip-G1 reference** (`services/video-worker/harness/tests/test_reference_g1.py`): reproduces the oracle. Time base, tags and all 2,209 timestamps identical; 1,587 audio packets identical; 21 of 21 activity events; onset frames identical for 16 of 16 blocks; pan error under 0.8 px; and it reports the reference's one on-screen shortfall (B2022, 1.05 px against a 1 px limit) as a failure.

**The product's own clip-G1 output** (`tests/test_g1_onscreen.py`), measured black-box by the same harness:

| Gate | Result |
| --- | --- |
| Blocks found and modelled | 16 of 16 reference blocks |
| Pixels changed outside the declared mask, before encoding | 0 in every frame |
| PSNR outside the mask after encoding | above 45 dB in every segment |
| Timestamps, frame count, time base, container aspect ratio, colour tags | identical to the source |
| Audio packets | identical, the last one and its duration included |
| Activity events | match the source frame for frame |
| First-visible and fully-visible frames | identical to the source on all 16 blocks |
| Opacity during reveals | within 0.05 on all 16 blocks |
| Offset during reveals | within 1 px on all 16 blocks, B2022 included |
| Position during pans | within 1 px |
| Numerals and neighbouring objects intact | **fails on 1 of 16 blocks** (see below) |

Wall-clock for the deterministic stages on clip-G1: about 3 minutes (analysis, 16 block models, render and encode), against a 10-minute target that also has to include the agent's turns.

**Synthetic clip-G15** (`tests/test_g15_onscreen.py`, `tests/test_tools.py`): onsets, slide offsets and camera travel match what the generator drew; rebuilt background within 0.5 grey levels on average of the true text-free scene; at most 2 stray pixels in any frame; changing one block re-encodes only the segments it is in, proven by segment hashes.

**Perturbed clip-G1** (`tests/test_g1_perturbed.py`): crop, pad and time-shift pass (every block's ink box, onset frames, size and every pan move by the known transform). **Rescale and re-encode fail**: on the rescaled copy one block's first visible frame is found one frame late; on the re-encoded copy one caption's text area takes in part of a connector ring next to it. Both are real robustness gaps.

**API and web**: 15 API tests (`apps/api/test/vt-*.test.ts`) and 14 Playwright specs (`apps/web/e2e/video/`) pass. The agent tests replay recorded model output and a fake worker and say so in every event. The web specs run against a mocked API. One real-stack check was done by hand: sign up, chunked upload, probe through the real worker, start a translation, and the job failing with the missing-key message.

## Known failures and gaps

**From the real clip-G1 run** (the agent's own closing list, confirmed where checked):

- Took 16 minutes against a 10-minute target. A good part went on rework: tool inputs with Hindi written as escape codes arrived garbled several times before the agent typed the characters directly (the prompt now says to), and a late regrouping wiped finished work (regrouping now keeps unchanged blocks).
- The agent saw the damaged connector ring beside the 1962 caption in its own zoomed review and worked around it by splitting that caption into two blocks; two tiny marks from the old text remain under its second line, and its last line overflows by 1 px.
- A faint fragment of English shows after one caption during a pan (frames 1291 to 1299).
- One heading is drawn lighter than the source heading, and one bullet was shortened to fit.
- The leftover-text check failed on three blocks where the "leftover" is digits the Hindi keeps (1991, 20,000, 2018). That is a false alarm in the check, which should ignore text the translation itself contains.
- The first attempt failed after ten minutes with a request-too-large error, and a naive fix (dropping old pictures) broke the model's signed thinking. Pictures are now uploaded once through the Files API and referenced by id, which keeps the conversation append-only.

- **F1.3 fails on clip-G1**: beside the 1962 caption, the ring at the end of a connector line is partly erased (about 86 pixels), because it is the size and colour of a letter and touches the line of text. `test_F1_3_no_numeral_or_neighbouring_object_is_partly_erased` fails and stays red until this is fixed. The other 15 blocks are clean.

- **F2.2**: the rebuilt background's 99th-percentile error reaches 3.8 grey levels on pan frames of clip-G15; the limit is 3.
- **Colour-on-colour text** (clip-G15 colour fringe): saturated text on a saturated panel is found, but its measured ink colour is washed out by 4:2:0 chroma and one of the two blocks is not modelled at all.
- **Occluders** (clip-G15 occluder): a block that something passes in front of is not modelled yet; where a frame does not match the text model the frame is left as in the source and reported.
- **Bitstream aspect ratio and rotation** are not yet checked (container values are).
- **Voiceover half of the Phase 0 gate**: the loudness, true-peak, click and back-transcription detectors are not built, so the reference's two voiceover shortfalls (I4.2 on S5, I3.2 on S7d) are not yet reproduced.
- **Remnant OCR in the harness** is not independent yet: the product's own read-back runs in `qa_suite`; an independent engine (Tesseract in CI) is still to add.
- **Font weight** is matched by stroke thickness against Noto Sans Devanagari only. Poppins, Hind and Mukta are not vendored yet, so there are no font alternatives to offer.
- **Layout**: with a 1.33 em line pitch, two clip-G1 blocks (2012 and 2022) overflow their panels with the reference wording. The agent is expected to reword; that has not run.
- **Uploaded files live on the API's disk**, because the worker reads by path. API and worker must share a disk for now.
- **Pre-existing**: `npm run typecheck` fails in `packages/loc-formats/test/zip-dir.test.ts` on `dev`, unrelated to this work.

## Cost and speed of a video job (measured, clip-G1, 74 seconds of video)

| Run | Model turns | Pictures sent | Time | Model cost |
| --- | --- | --- | --- | --- |
| First real run (one call per caption, high effort) | 41 | 138 | 16 min | $2.99 |
| After batching and medium effort | 20 | far fewer (sheets) | 7 min, including about a minute waiting on a question | $1.11 |

Cost is from the token usage stored on every model turn, at Opus 5.5 list prices ($4 in, $20 out, $0.20 cache read, $5 cache write per million tokens). That is about $0.90 per minute of video now, down from $2.44. What changed: the agent models all captions in one call, checks all translations in one call and previews several frames on one sheet, so it takes half as many turns and re-reads its context half as often; effort defaults to medium. Further savings not yet taken: smaller pictures for routine checks, a cheaper model for the mechanical turns, and skipping the per-block zooms when the measured checks are clean.

In the second run the agent asked the user a real question (keep the 1962 caption in English, or translate it and damage the connector ring) and followed the answer. 7 of 8 of its checks passed; the failing one was the leftover-text false alarm, fixed afterwards.

## Localization file types (browser journeys)

`apps/web/e2e/zz-loc-asset-matrix.spec.ts` takes 26 file types through upload, Workbench, translate, the translation view and export in a real browser: Word, Excel, PowerPoint, PDF, Markdown, CSV, SCORM 1.1 / 1.2 / 2004 and a media-rich SCORM, HTML, XLIFF 1.2 and 2.1, JSON, gettext, Java properties, .NET resx, Flutter ARB, YAML, iOS strings / stringsdict / xcstrings, SRT, VTT, MP4 and MP3. All 26 pass locally against the scripted translator. Files that are not web pages now have a side-by-side reading view (they used to show an empty message). The recordings in the fixtures are two seconds long with no speech, so media is covered for upload, player and export only.

## Voice dubbing (Phase 3, first cut)

**What ran for real:** nothing against ElevenLabs. There is no ElevenLabs key in the environment, so no speech has been transcribed, separated or synthesized by this code. A dub has never been heard.

**What is built and tested without a provider**

- Worker audio (`services/video-worker/vw/audio/`): phrase bounds refined on the audio, loudness and true peak, pitch-preserving stretch held to 0.90 to 1.10 of natural speed, background rebuilt from the original minus the isolated voice, mixing on the source onsets with per-phrase and whole-track loudness matching and a -1 dBTP limit, and a mux that copies the video packets untouched (replace the voice, or keep the original and add the dub as a second track). Tested on synthetic audio with known answers: 17 tests.
- The harness half Phase 0 owed (`harness/audio/`): it measures the hand-made clip-G1 dub and reproduces the oracle, including the phrase the reference gets wrong (S7d, 16.9 % short). The other shortfall (S5, a word spoken 1.26 s late) needs word timings from a real transcription and is marked blocked.
- API (`apps/api/src/video/voice/`): a provider interface with one ElevenLabs implementation, key taken from the project owner's saved key and then the server's, usage counted per version, seven agent tools, and routes to reword one phrase, change a speaker's voice and choose how the audio is carried. Tested against a fake ElevenLabs server and the replay worker: 7 tests.
- Web: Voiceover switch and voice picker on Upload, an ElevenLabs card in Settings (key, default voice, voice per language), a voice lane and phrase editor in Review, audio choices in Export. Tested in a browser against a scripted API: 5 specs.
- Checks: when a version has a dub, the worker's `qa_suite` adds onset, duration, speed, anchor, loudness, true peak, clicks, audio length, untouched video and key-term checks. The key-term check fails until the dub has been transcribed back.

**Waiting on an ElevenLabs key:** every claim about real speech. First real run should be clip-G1's narration; expect the same kind of surprises the first on-screen run had.

**Not built:** lip-sync (off, labelled "Later"), voice cloning and its consent record, a stored pronunciation dictionary, editing the transcript before translation, matching the default voice to the speaker's pitch, captions-only export, keeping laughs and breaths, fitting into the speaker's own pauses.

**To run a real dub:** Settings → Models → ElevenLabs API key (or set `ELEVENLABS_API_KEY` on the API and worker services), then switch Voiceover on when starting a translation.

## Real dubbing run and data clean-up (2026-10-08)

- **A real dub exists.** clip-G1 with Hindi on-screen text and an ElevenLabs Hindi voiceover completed on the local stack: 16 of 19 checks, $2.20 model cost, about 2,100 ElevenLabs characters. The three failures, all reported by the agent: one phrase starts 254 ms late because two phrases overlap in the source audio; every phrase is about 1.1 LU quieter than the one it replaces; one key term was heard but spelled differently when transcribed back. Nobody has listened to it yet.
- **It took four attempts.** Found and fixed: the API allows 20 strict tools and a request carried 23 (every video job failed until fixed); the model ran away with its output writing a large batch of Hindi in one strict call (a cut-off step is now retried in smaller parts, long-text tools are no longer strict, output per step is capped). Not explained: the job's lease was lost once during the final render; resuming completed it.
- **The agent cannot tell a speaker's gender**, so it asks. A pitch measurement in the worker would let it choose.
- **Old on-screen results were deleted from the dev database** at the owner's request: 167 segments, 4 documents and the on-screen state of 60 assets, all in the LOC-DEMO project. A JSON backup was written first. The rendered video files remain in object storage. Production was not touched.
- **Dubbed videos in course packages**: an exported package again carries the dubbed voice on its video, with the picture stream copied untouched.

## Dubbing fixes after the owner listened (2026-10-08)

The owner heard the English voice under the Hindi in the first real dub. Measured afterwards on that file: the source voice sat 20.5 dB under the new one (the limit is 30), because the background kept under the dub was "the original with the voice masked out", and the mask left part of the voice in.

What changed:

- **Separation decides what to keep.** Music, effects and ambience are kept with a spectral soft mask that also holds any leftover voice at least 34 dB down; when there is only room noise or silence between phrases, nothing of the old track is kept (generated room tone, or nothing). clip-G1 is narration over silence, so its dub now contains no part of the source track.
- **A check for it** (`voice_residual`, story I2.2) fails a dub whose kept background still follows the source voice above -30 dB. A kept background is also transcribed and must contain no words.
- **Loudness**: per-phrase and whole-track gains are balanced, so both pass together.
- **Overlapping source phrases** of one speaker are merged into one unit before dubbing; a phrase is never cut to make room.
- **Key terms** are matched across scripts, spellings and number words.
- **Pitch** is measured per speaker, so the agent picks a voice without asking.
- **Waveform peaks** are available for the editor's voice lane (data only; the lane does not draw them yet).

Second real clip-G1 dub (`docs/demos/phase-3/real-run/Mahindra_timeline_hi_dubbed_v2.mp4`): 20 of 20 checks; largest onset error 25 ms; every phrase within 0.4 LU of its source phrase; 9 of 9 key terms heard; no question asked (265 Hz, a female library voice chosen); 9 min 47 s; $1.71 model cost, 1,948 ElevenLabs characters, 221 s transcribed, 74 s isolated. In the file, what is not the new voice measures -44.6 dBFS while the source voice would be speaking and -48.0 dBFS elsewhere (encoder noise); in the first dub those were -36.0 and -51.2. `bleed_before_after_spectrogram.png` beside it shows the difference. **Nobody has listened to the new file yet.**

Still open: the voice-over-music path is proven only on synthetic mixes; the provider's own stem separation is wired but never run against the live service; anchor timings in that run are estimates from word position (the six mentions were not individually timed from the transcription); the agent condensed some captions to fit.

## Independent review (Phase 1)

A separate reviewer reran the suites and inspected the clip-G1 output; its report is [`reviews/phase-1.md`](./reviews/phase-1.md) with evidence images beside it. Verdict: **the Phase 1 gate is not met.** Its reruns matched the numbers above.

Fixed after the review: only the first audio track survived export (all tracks are now copied, checked on a two-track file); the trace would have printed "0 blocks drawn"; a job could close clean without exporting or checking anything; the pixel test asserted a constant instead of rendering; the checks tool described checks it does not run.

Still open from the review:

- The product's own `qa_suite` has no check for damaged neighbouring objects, onsets or motion. Those are measured only by the test harness, so a real job would not have caught the damaged ring itself.
- A cold render of clip-G1 produced different video packets from the cached one; the cause is not found, and the segment cache key omits some compositor inputs. Until this is understood, "identical segment hashes" proves reuse, not reproducibility.
- The background-error test asserts 4.0 grey levels against a story limit of 3 (recorded as a FAIL in traceability), and the tool test leaves out the layout check, which fails on purpose-built overflowing text.
- Remnants on clip-G1 are not measured pixel-by-pixel by any test (only on the synthetic clip).
- Only Devanagari and Latin can be drawn; other scripts fail with an unhelpful error.
- API 500 responses return the raw error message, and the worker has no authentication (it listens on localhost only).

## Blocked on you

| Need | Blocks |
| --- | --- |
| A spend limit for the Anthropic key (the key is in `.env` now) | Running the golden set repeatedly |
| Decision on x264 (GPL, bundled in the PyAV wheel) and H.264 patent terms, or an alternative encoder | Shipping exports commercially |
| Golden clips G2 to G14 with rights, and human labels | Most of Phases 3 and 4 |
| GPU host and vendor choices for voice, separation and lip-sync | Phase 3 |
| A native Hindi reviewer, or approval of agent review as an interim | Criteria that need a human |

The MongoDB connection string and OpenRouter key pasted into this session's chat should be rotated.

## How to run it locally

```
# worker (Python 3.12 via uv; no ffmpeg binary needed)
cd services/video-worker && ./run.sh                         # http://127.0.0.1:8791
# API: use a throwaway file datastore and local disk; never the shared .env's database or bucket
OBJECT_STORE=local DB_BACKEND=file DATA_DIR=$PWD/.tmp/video-dev PORT=8080 ANTHROPIC_API_KEY=... npm run dev:api
# web
npm run dev -w @teststudios/web                               # then open /?page=video

# tests
cd services/video-worker && uv run pytest -q                  # harness + pipeline (about 15 minutes with clip-G1)
node --import tsx --test apps/api/test/vt-*.test.ts
npx playwright test -c apps/web/e2e/playwright.video.config.ts
```
