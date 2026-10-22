# Plan: Video Translate

The build plan for the product described in `PROMPT.md`: upload any video, get it back in another language with on-screen text replaced in place, speech dubbed on the original timeline, and nothing else changed. The 122 stories in `handoff/USER_STORIES.md` and `handoff/STORIES_ADDENDUM.md` are the source of truth; `docs/TRACEABILITY.md` maps each criterion to its test. `docs/AUDIT.md` says what exists today.

Decisions already made by the owner:

- **Model:** Opus 5.5 through the Claude API directly (`claude-opus-5-5`).
- **Scope:** all six phases, back to back.
- **Look:** Vozo's layout and flow with the KaDeep orange accent and our own name and assets.
- **Compute for now:** this Mac, CPU only (an assumption; the owner did not pick an option).

## 1. Architecture

Three parts, each with one job.

```
Browser (apps/web/src/video)          Node API (apps/api/src/video)             Python worker (services/video-worker)
  Upload · Working · Review · Export    Opus 5.5 agent loop, stage graph,          deterministic tools: decode, measure,
  frame-indexed player, trace list  ⇄   artifact store, trace, routes        ⇄    rebuild, shape, composite, encode
                                        owns the job lease (Mongo queue)           HTTP sidecar, no queue access
```

### Python worker (`services/video-worker/`)

- A FastAPI sidecar in the same pattern as `services/memory-graph`, Python 3.12 managed by uv.
- **Interface:** `POST /tools/{name}` with `{params, inputs:[sha], source:{sha, uri}}` returns `{task_id}`; `GET /tasks/{id}/events` streams NDJSON progress with real counts; `POST /tasks/{id}/cancel`; `GET /health` reports loaded models and tool versions.
- **Node owns the lease.** The worker never touches the queue or the database. This avoids reimplementing lease semantics in a second language, and the local JSON store cannot be shared across processes anyway.
- **Stack:** PyAV (bundled FFmpeg), NumPy, OpenCV, SciPy, uharfbuzz + freetype-py for shaping, RapidOCR on onnxruntime for text. No PyTorch until Phase 3.
- **Frames:** for clips up to about five minutes, one sequential decode into a memory-mapped file of native planes; for longer clips, per-GOP decode on demand against the frame index with a small cache. Tools take frame ranges and never hold more than a chunk in memory.
- **Concurrency:** a process pool of three on the 16 GB development machine.
- **Models are baked in and kept warm.** Nothing is downloaded during a job.

### Orchestrator (`apps/api/src/video/`)

- An Opus 5.5 loop on `@anthropic-ai/sdk`: streaming, adaptive thinking with summarised display, an explicit effort per stage, strict tools, parallel tool calls answered in one message, prompt caching on the system prompt, tools and video context.
- **The agent decides; it does not push pixels.** It plans per shot, confirms and groups text, classifies blocks, writes translations, resolves layout conflicts, reviews zoomed renders, and turns user instructions into edits. Everything else is a deterministic tool.
- **Append-only conversation.** Turns are stored verbatim (`video_agent_turns`) with images as content-hash placeholders and re-inflated to identical bytes on resume. Earlier turns are never edited, because that invalidates thinking blocks.
- **One artifact store for the model and the UI.** A per-organisation content-addressed store (`orgs/<orgId>/video/cas/<aa>/<sha256>`), added beside the existing key helpers in `apps/api/src/services/object-store.ts`. An image sent to the model and the image shown in the trace are the same object with the same hash. It is scoped per organisation so that deduplication cannot reveal that another workspace holds a file.
- **Jobs:** a new `video_translate` kind on the existing queue (`apps/api/src/jobs/queue.ts`), run by `apps/api/src/jobs/video-worker.ts`, modelled on `loc-worker.ts`. Pause and cancel are checked at every tool boundary and forwarded to the worker. Questions park the job as waiting for a human.

### Stage cache and segments

- **Stage cache** (`video_stage_cache`): key `sha256(tool | toolVersion | canonical params | sorted input hashes | source hash)`, value `{resultSha, artifactShas[], ms}`. The worker computes the key because it owns the tool version.
- **Segments are fixed closed-GOP chunks of about 60 frames that never straddle a cut.** Not per shot: clip-G1 is a single shot, so per-shot segments would make a partial re-render impossible.
- **Segment key:** `sha256(source hash | frame range | canonical render specs of the blocks that intersect the range, including their per-frame track slices | compositor version | encoder settings)`. An edit to one block changes the keys of only the segments it touches; story K4 is proven by key equality on the rest.

### Pixels, time and audio (rules 2.1–2.3)

- **Composite in the source's native Y′CbCr planes.** A whole-frame round trip through RGB changes pixels everywhere and breaks the zero-pixel rule. Only the patch under the mask is converted (with the source's matrix and range), blended with premultiplied alpha, converted back and written with a masked copy.
- **The declared change mask is chroma-aligned:** the ink mask is projected to chroma resolution (any chroma sample whose footprint touches ink is included) and expanded back to luma. The compositor asserts in-process that no sample outside it differs before anything is encoded.
- **Encode** per segment with libx264 as closed GOPs with identical parameter sets, using the source's own PTS, time base, sample aspect ratio (bitstream and container) and colour tags. After each segment, PSNR outside the mask is measured and the segment is re-encoded at higher quality if it is below 46.5 dB. The reference's own minimum was 46.2 dB against a 45 dB gate, so the margin is thin and is checked, not assumed.
- **Audio that is not translated is stream-copied packet by packet to end of file.** No shortest-stream option exists anywhere in the code. After export, packet count, bytes, timestamps and the final packet are compared with the source.

### Player and editor (`apps/web/src/video/`)

- **The browser never decodes the source.** WebCodecs cannot decode ProRes, and HEVC depends on hardware. At ingest the worker builds a **review proxy**: H.264, one access unit per source frame, short GOPs, no B-frames, with a sidecar index `{i, srcPts, byteOffset, size, key}`. HDR sources get a tone-mapped proxy, labelled as such.
- Both panes decode the proxy with mp4box.js and WebCodecs onto canvases. **Frame identity is the index, never `currentTime`.** One clock owns the frame index; both panes present frame N in the same animation frame only when both have it, and otherwise both hold. Audio is the playback master, mapped to an index through the PTS table, which keeps variable frame rate correct.
- A server PNG frame endpoint is the fallback for stepping.
- The translated pane shows rendered proxy segments as they land; unrendered ranges say so. An edit preview is a patch drawn over the frame, replaced by the re-rendered segment.

### Trace

- **Schema** (`packages/shared/src/video-trace.ts`): an envelope `{seq, jobId, orgId, versionId, at, rowId, parentRowId?, op, row}` with row kinds thinking, action, image, result, question and stage.
- **Storage:** `video_trace_events`, unique on `{jobId, seq}`. The lease holder is the only writer; a stale worker's insert collides and it stops. Writing never blocks the job.
- **Delivery:** `GET /api/video/jobs/:id/trace` as SSE, honouring `Last-Event-ID`, replaying then tailing. Socket.IO only nudges, so it works across replicas.
- **Redaction** at a single publishing choke point, before anything is persisted: a per-kind field allow-list, scrubbers for secrets, key patterns, internal paths, bucket keys and system-prompt text, and an ownership check on every artifact. "Steps only" is a read-time filter.
- **Images** are normalised to at most 2,576 px on the long edge before hashing (a configuration value, since it is model-specific), with a WebP thumbnail of at most 60 KB.
- The web list is a new pure reducer and a virtualised `TraceList.tsx`, written in the style of the existing chat timeline and reusing its classes.

## 2. Stages

Shown to the user as the stage bar of story C1.

| # | Stage (user wording) | What happens | Decided by |
| --- | --- | --- | --- |
| 1 | Read video | Probe, frame index with exact PTS, review proxy, repair or reject damaged files | Tools |
| 2 | Map motion | Cuts and dissolves, activity map, measured per-frame camera track, shot types | Tools; the agent labels shots and writes the plan |
| 3 | Find text | Detection and OCR on settled frames, per-frame tracks, first/fully/last visible frames | Tools; the agent confirms, groups, links and classifies |
| 4 | Translate | All blocks, the transcript and the visuals together; glossary, style guide, number check, fit to space | The agent; layout checker is a tool |
| 5 | Remove text | Ink masks including anti-aliased edges and chroma fringe; protected objects segmented | Tools |
| 6 | Rebuild background | Clean plate → layer model → temporal fill → generative fill (gated, labelled) | Tools pick the cheapest method the evidence supports; the agent approves escalation |
| 7 | Animate | Per-frame opacity, offset, scale, rotation, blur and reveal transferred from the source track | Tools |
| 8 | Voice | Stems when needed, speech recognition, speakers, timed translation, speech, lip-sync, mix | The agent writes the timed translation; the rest is tools and providers |
| 9 | Render | Composite inside the mask, encode by segment, copy untouched audio | Tools |
| 10 | Check | The full QA suite, then the agent's zoomed review of risky spots, up to three fix rounds | Tools measure; the agent reviews |

Export (formats, sidecars, QA certificate) follows on request.

## 3. Agent tools

Each returns JSON plus optional image artifacts; every artifact goes to the trace.

| Area | Tools |
| --- | --- |
| Reading the video | `probe`, `frame_index`, `extract_frames`, `contact_sheet`, `crop_zoom` |
| Shots and motion | `shot_detect`, `shot_classify`, `activity_map`, `camera_motion`, `track_points` |
| Text | `detect_text`, `ocr`, `track_text`, `group_blocks` |
| Scene and appearance | `segment_objects`, `measure_typography`, `estimate_blur_lighting_grain` |
| Background | `clean_plate`, `layer_model_fit`, `temporal_fill`, `generative_fill` (gated) |
| Layout and rendering | `layout_fit_check`, `render_text_preview`, `transfer_animation`, `composite_shot` |
| Checking pictures | `diff_map`, `remnant_scan`, `qa_suite` |
| Audio | `separate_stems`, `asr`, `diarize`, `active_speaker`, `estimate_duration`, `tts`, `voice_clone` (gated), `lip_sync` (gated), `mix_audio`, `back_transcribe` |
| Output | `encode_export` |

Methods, in brief (generalised from the clip-G1 report; nothing here knows clip-G1):

- **Camera:** coarse translation by phase correlation on a gradient image with periodic structure suppressed, a feature check to resolve aliasing, then per-frame refinement against a mosaic. Measured tracks, no velocity model.
- **Text:** OCR only on settled frames (one per second plus the first quiet frame after each event), small text upscaled before recognition.
- **Onsets:** per block, a robust difference against the pre-appearance plate; thresholds are calibrated on the synthetic suite only, never on clip-G1.
- **Layer model:** per channel `C = a·P + b` by robust least squares over the container.
- **Animation:** per intro frame, fit `frame ≈ β·box + α·shift(ink, dx, dy)` and use the measured values directly.
- **Shaping:** HarfBuzz at 4× supersampling, with a keep-out checker dilated by 3 px that runs before every render.

## 4. Data model

All on the existing `Store<T>` layer. New collections only; nothing existing changes (see `docs/AUDIT.md` §6).

| Collection | Holds |
| --- | --- |
| `video_sources` | Probe data, the frame index with PTS, the proxy, the file hash |
| `video_analyses` | Shots, camera tracks, text tracks and blocks, protected objects, speakers, phrases, visual anchors |
| `video_versions` | One per target language: translations, layout, animation, voice assignments, manual locks |
| `video_renders` | Segments with their keys and hashes, change masks |
| `video_qa_reports` | Check results, per-stage time and cost |
| `video_stage_cache` | Tool results by content hash |
| `video_agent_turns` | The verbatim agent conversation |
| `video_trace_events` | The trace |
| `video_edit_log` | Each AI change: what, where, why, resulting version |
| `video_consents` | Voice-clone and lip-sync confirmations, rights confirmations |

Workspaces, projects, the glossary, the audit log and the usage ledger are the existing ones.

## 5. Phases

Each phase ends with its tests green in CI, an independent reviewer agent's report in `docs/reviews/phase-N.md`, a Playwright recording in `docs/demos/`, and `docs/STATUS.md` updated. Work is committed locally on `video-rebuild`; nothing is pushed or deployed.

| Phase | Scope | Gate |
| --- | --- | --- |
| **0** | This audit and plan, `MODELS.md`, traceability, the measurement harness, the clip-G15 synthetic generator and the perturbed-G1 generator | The harness, run on the clip-G1 reference outputs, reproduces the oracle's results and reports its three shortfalls (I4.2 on S5, I3.2 on S7d, H3.1 on B2022) as failures |
| **1** | Vertical slice: upload → analysis → on-screen translation → export, with the live trace, on clip-G1, perturbed-G1 and clip-G15 | Every applicable release-gate row passes on the product's own output; no clip-G1-specific code; the trace shows real work |
| **2** | Review and editor: frame-accurate player, compare modes, layer timeline, tabs, canvas edits, versions, select-and-tell | J1–J9 and K1–K8 pass; one edit on clip-G1 re-renders only the touched segments in ≤ 60 s |
| **3** | Voice: clip-G1's narration, then clip-G2, then clip-G9 with lip-sync | I1–I9 and V5–V9 pass on their clips; clip-G1's dub beats the reference's two voice shortfalls |
| **4** | Any-video breadth: live action, occlusion, kinetic type, screen recordings, VFR and portrait, 4K/HDR long form, RTL and other source languages | All V stories pass on their clips; failures are flagged honestly |
| **5** | Speed, reliability, cost, privacy, roles, sharing, QA certificate, accessibility, polish; removal of the old pipeline and studio | A full real-model release run is green on every applicable clip–story pair; old code removed after confirmation |

## 6. Speed budget

Story N1: ≤ 10 minutes for a ≤ 2-minute 1080p motion-graphics video with ≤ 20 blocks, ≤ 15 minutes with voice, first preview ≤ 3 minutes, 95th percentile ≤ 1.5× these. The hand-run reference took about 61 minutes for on-screen text.

The budget below is a target to measure against, not a measurement. Each stage's real time and cost go into the QA report and the admin view (N4) from the first run.

| Stage | Budget | How |
| --- | --- | --- |
| Read video, proxy, frame index | 0:30 | One sequential decode; proxy encoded in parallel |
| Map motion | 0:45 | Luma-only pass; camera refinement only on moving frames |
| Find text | 1:00 | OCR on tens of settled frames, not every frame |
| Agent: plan, confirm, classify | 1:00 | Sampled and zoomed frames; prompt caching; low effort for routine turns |
| **First preview** (one block, before/after still at 1:30, playable by 3:00) | by 3:00 | The first finished block renders ahead of the rest |
| Translate and fit | 1:30 | One pass over all blocks, parallel fit checks |
| Remove, rebuild, animate | 1:30 | Per block in parallel |
| Render | 1:30 | Segments in parallel |
| Check and self-review | 1:30 | Suite in parallel; up to three fix rounds only where something failed |
| **Total, on-screen only** | **≤ 10:00** | |
| Voice (when on) | + 5:00 | Speech recognition, timed translation, speech, mix, back-transcription |

Model latency is expected to be the largest single cost. On this CPU-only Mac the target may not be met; the number will be reported as measured, on the machine it was measured on.

### Proposed turnaround targets for other video kinds (V12.5)

**Proposed, to be confirmed by the owner after measurement.** N1 covers only short motion graphics. These are starting proposals and none has been measured yet.

| Video | Proposed target | First preview |
| --- | --- | --- |
| 3-minute 1080p talking head, voice only | ≤ 12 min | ≤ 3 min |
| 3-minute 1080p talking head with lip-sync, one GPU | ≤ 25 min | ≤ 4 min |
| 3-minute two-speaker interview with lip-sync and music | ≤ 30 min | ≤ 4 min |
| 10-minute 4K60 webinar with slides, on-screen text only | ≤ 30 min | ≤ 5 min |
| 60-minute 1080p recording, on-screen text and voice | ≤ 2 × duration (120 min) | ≤ 10 min |
| Partial re-render of one block in a 2-minute 1080p video | ≤ 60 s (K4, already fixed) | preview ≤ 30 s |

## 7. What is needed from the owner

One list, asked once (brief §11). Everything that does not depend on an answer keeps moving; anything waiting is marked BLOCKED in `docs/STATUS.md` and is never passed with a stand-in.

| # | Need | What it blocks | Meanwhile |
| --- | --- | --- | --- |
| 1 | `ANTHROPIC_API_KEY` in `.env`, and a spend limit | Every real agent run from Phase 1 on | Phase 0 in full; the orchestrator is built against recorded replays, labelled as replays |
| 2 | A GPU host (type, count, region), and where object storage and the database for staging live | Lip-sync and 4K/HDR speed in Phases 3–4; any speed claim beyond this Mac | CPU paths and provider adapters |
| 3 | Vendor choices and keys: speech (ElevenLabs is already wired), voice cloning, lip-sync, stem separation | Phase 3 gates | A licence and cost comparison is in `docs/MODELS.md` for the choice |
| 4 | Golden clips G2–G14 with rights to use them as test material | Most of the Phase 3–4 gates | We can propose openly licensed clips that allow commercial use and record each licence; the synthetic suite covers what it can |
| 5 | Human ground-truth labels for those clips (text blocks, cuts, speaker turns, transcripts, defects) | The same gates; labels must come from people | A small labelling tool; agent-proposed labels count only after a person checks them |
| 6 | Launch languages | A shaping test sheet per script (V13.2) | Hindi (Devanagari) and Latin first |
| 7 | Confirmation of the proposed turnaround targets above | V12.5 | Measured numbers are reported against the proposals |
| 8 | Native reviewers for languages where a criterion needs one, or approval to use agent review as an interim, labelled measure | Criteria on meaning and prosody | Status AGENT-REVIEW-ONLY, which is not a pass |
| 9 | A decision on x264 (GPL, or a commercial licence) and H.264/HEVC patent terms | Shipping exports commercially | Development proceeds; see `docs/MODELS.md` |
| 10 | Whether payments are in scope | Credits, plans and the free-tier watermark (C5.1, N4, L4) | Built against the existing usage ledger |
| 11 | Installing Homebrew, ffmpeg and tesseract on this Mac, or leave as is | Running the independent OCR check locally | PyAV's bundled FFmpeg is used; the OCR check runs in CI |
| 12 | Confirmation of the final removal list before the old pipeline and studio are deleted | Phase 5 close | The list is in `docs/AUDIT.md` §4 |

Two housekeeping notes for the owner: the MongoDB connection string and the OpenRouter key pasted into the planning session should be rotated, and neither is used by this work.
