# Video Translate: internal contracts

Three parties: the Python worker (deterministic tools), the Node API (accounts, jobs, the agent, the trace) and the
web app. This file is the contract between them. Change it first, then the code.

## 1. Worker (Python, `services/video-worker`, default `http://127.0.0.1:8791`, env `VIDEO_WORKER_URL`)

The worker reads files from a path the API gives it and never talks to the browser.

| Call | Body | Returns |
| --- | --- | --- |
| `GET /health` | | `{ok, version, tools: string[]}` |
| `POST /sources` | `{path}` absolute path of the uploaded file | `Source` |
| `POST /sources/:sid/tools/:tool` | `{session, params}` | `ToolResult` |
| `GET /artifacts/:sha` | | image bytes (`image/png`, `image/webp` or `image/jpeg`) |
| `GET /sources/:sid/frames/:index.jpg?session=&view=source\|translated` | | one frame as JPEG |

`sid` is the SHA-256 of the file. `session` names one translation in progress (one target language of one
project); the worker keeps that session's blocks and wording.

```ts
type Source = {
  source_id: string; frames: number; width: number; height: number; fps: string; duration_s: number;
  time_base: string; variable_frame_rate: boolean; pix_fmt: string; video_codec: string;
  audio: { codec: string; sample_rate: number; channels: number; language?: string }[];
  color: { range: string|null; space: string|null; primaries: string|null; transfer: string|null };
  sar: string|null; rotation: number; oddities: string[];   // plain-word notes, e.g. "variable frame rate"
};
type Artifact = {
  sha256: string; thumb_sha256: string; kind: ArtifactKind; title: string; legend: string;
  width: number; height: number; bytes: number; frame?: number; frame_range?: [number, number];
  timecode?: string; produced_by: string; file_name: string;
};
type ArtifactKind = 'contact_sheet'|'zoom_crop'|'mask_overlay'|'diff_map'|'before_after'|'camera_path'
  |'activity_map'|'layout_check'|'frame'|'waveform'|'face_pair';
type ToolResult =
  | { ok: true; tool: string; result: Record<string, unknown>; artifacts: Artifact[]; ms: number; cache_hit: boolean }
  | { ok: false; tool: string; error: { code: string; message: string } };
```

Images are at most 2,576 px on the long edge before hashing; thumbnails are WebP of at most 60 KB.

Tools available now (on-screen text). Every result is JSON; numbers in it are measured, never estimated.

| Tool | Params | Result (main fields) |
| --- | --- | --- |
| `probe` | | the `Source` |
| `contact_sheet` | `{start?, end?, per_second?}` | `{sheets: [{frame_range, frames: number[]}]}` + one image per sheet |
| `crop_zoom` | `{frame, box: [l,t,r,b], scale?, title?}` | `{frame, box}` + the crop with a highlight box |
| `analyse` | | `{segments, events: [{start,end,kind,coverage}], pans: [{start,end,dx,dy}], noise}` + activity map, camera path |
| `detect_text` | | `{lines: Line[], blocks: Block[]}` + one overlay per page |
| `set_blocks` | `{blocks: [{id, line_ids, kind, reason?}]}` | `{blocks: Block[]}` |
| `model_block` | `{block_id}` | `{typography, panel, first_visible, fully_visible, last_visible, notes, fit_error}` + mask overlay, before/after |
| `layout_fit_check` | `{block_id, lines: string[], script}` | `{ok, overflow_right, overflow_bottom, keepout_hits, line_overlaps, notes}` + layout overlay, preview |
| `render_preview` | `{frame}` | `{frame, blocks_drawn}` + before/after |
| `export` | `{}` | `{path, segments: [{index,start,stop,sha256,reused,crf}], changed_outside_mask, psnr_outside_mask_min_db, frames_left_untouched_by_guard}` |
| `qa_suite` | `{}` | `{checks: [{id, story, title, pass, value, limit, detail}]}` + difference maps |

```ts
type Line = { id: string; text: string; box: [number,number,number,number]; score: number; kind: 'text'|'numeral';
  first_frame: number; last_frame: number };
type Block = { id: string; line_ids: string[]; text: string[]; box: [number,number,number,number];
  kind: 'translatable'|'logo'|'numeral'|'in_photo'|'ui'|'watermark'|'real_world'; reason: string;
  first_frame: number; last_frame: number; ref_frame: number; screen_box: [number,number,number,number] };
```

Boxes in `Line`/`Block.box` are world coordinates (they do not move when the camera pans); `screen_box` is where
the block sits on `ref_frame`.

## 2. API (Node, `/api/video/*`, existing auth: `Authorization: Bearer <access token>`)

| Call | Body | Returns |
| --- | --- | --- |
| `POST /api/video/uploads` | `{name, size, sha256?}` | `{uploadId, chunkSize, duplicateOf?: projectId}` |
| `PUT /api/video/uploads/:id/:index` | raw chunk | `{received}` |
| `POST /api/video/uploads/:id/complete` | `{rightsConfirmed: true}` | `VideoProject` |
| `GET /api/video/projects` | | `VideoProject[]` |
| `GET /api/video/projects/:id` | | `VideoProject` with `versions` |
| `POST /api/video/projects/:id/translate` | `{targetLanguages: string[], options: TranslateOptions}` | `{jobs: VideoJob[]}` |
| `GET /api/video/jobs/:id` | | `VideoJob` |
| `POST /api/video/jobs/:id/cancel` \| `/pause` \| `/resume` | | `VideoJob` |
| `POST /api/video/jobs/:id/answer` | `{questionId, optionId}` | `{ok}` |
| `GET /api/video/jobs/:id/trace?after=<seq>` | (SSE; also honours `Last-Event-ID`) | `TraceEvent` stream |
| `GET /api/video/jobs/:id/trace.json?after=&limit=` | | `{events: TraceEvent[], last: number}` |
| `GET /api/video/artifacts/:sha?thumb=1` | | image bytes (checked against the caller's org) |
| `GET /api/video/versions/:id` | | `VideoVersion` (blocks, wording, timing, QA) |
| `GET /api/video/versions/:id/frames/:index.jpg?view=source\|translated` | | frame JPEG |
| `GET /api/video/versions/:id/export` | | the exported file (`Content-Disposition` with `<title>_<language>.<ext>`) |
| `GET /api/video/versions/:id/qa` | | `QaReport` |

```ts
type VideoProject = { id: string; orgId: string; name: string; createdAt: string; source: Source & { sizeBytes: number };
  versions?: VideoVersionSummary[] };
type TranslateOptions = { onScreenText: boolean; voiceover: boolean; lipSync: boolean; captions: boolean;
  prompt?: string; glossaryIds?: string[] };
type VideoJob = { id: string; projectId: string; versionId: string; language: string;
  status: 'queued'|'running'|'awaiting_answer'|'paused'|'completed'|'failed'|'cancelled';
  stage: StageId; stages: { id: StageId; label: string; state: 'pending'|'running'|'done'|'skipped'|'failed';
    startedAt?: string; endedAt?: string; etaSec?: number }[];
  startedAt?: string; endedAt?: string; error?: string; lastSeq: number };
type StageId = 'read'|'motion'|'text'|'translate'|'remove'|'rebuild'|'animate'|'voice'|'render'|'check';
type VideoVersionSummary = { id: string; language: string; status: VideoJob['status']; jobId: string; number: number };
type VideoVersion = VideoVersionSummary & { projectId: string; blocks: (Block & { target?: string[];
  fit?: { ok: boolean; overflow_right: number; overflow_bottom: number; keepout_hits: number; line_overlaps: number; notes: string[] };
  typography?: { px: number; weight: number; pitch: number; left: number }; first_visible?: number; fully_visible?: number })[]; export?: { ready: boolean; sizeBytes?: number;
  segments?: unknown[] }; qa?: QaReport };
type QaReport = { passed: boolean; checks: { id: string; story: string; title: string; pass: boolean; value: unknown;
  limit: unknown; detail?: string }[] };
```

### Trace events

One ordered log per job. `seq` starts at 1 and has no gaps. Rows open, may be patched, and close.

```ts
type TraceEvent = { seq: number; jobId: string; at: string; rowId: string; parentRowId?: string;
  op: 'open'|'patch'|'close'; replay?: boolean; row: TraceRow };
type TraceRow =
  | { kind: 'thinking'; title: string; text: string }                       // title <= 90 chars; text as the API returned it
  | { kind: 'action'; tool: string; title: string; status: 'running'|'done'|'failed'; inputPlain?: string;
      outputPlain?: string; technical?: { params: unknown; ms?: number; cacheHit?: boolean } }
  | { kind: 'image'; title: string; legend: string; imageKind: ArtifactKind; sha256: string; thumbSha256: string;
      width: number; height: number; frame?: number; frameRange?: [number, number]; timecode?: string;
      viewedByAgent: boolean; producedBy: string; fileName: string }
  | { kind: 'result'; title: string; metrics: { label: string; value: string }[] }
  | { kind: 'question'; questionId: string; prompt: string; options: { id: string; label: string; description?: string }[];
      recommended: string; timeoutAt: string; answer?: { optionId: string; by: 'user'|'timeout' } }
  | { kind: 'stage'; stage: StageId; label: string; state: 'running'|'done'|'failed'; done?: number; total?: number; etaSec?: number };
```

SSE frames are `id: <seq>` / `event: trace` / `data: <TraceEvent JSON>`. Live thinking text between patches is sent
as `event: delta` frames without an id (`{rowId, text}`); the next `patch` carries the full text, so a reconnect
never loses or repeats anything.

## 3. Notes from the API implementation

- **Type names.** In `@teststudios/shared` these types carry a `Video` prefix, because `Artifact` and `ArtifactKind`
  already mean something else there: `VideoSource`, `VideoArtifact`, `VideoArtifactKind`, `VideoToolResult`,
  `VideoLine`, `VideoBlock`, `VideoProject`, `VideoTranslateOptions`, `VideoJob`, `VideoStageId`, `VideoStage`,
  `VideoVersion`, `VideoVersionSummary`, `VideoQaReport`, `VideoTraceEvent`, `VideoTraceRow`. The shapes are the ones
  above. `applyTraceEvent(state, event)` and `emptyVideoTrace()` fold events into ordered rows for the web.
- **Rows that happen at once.** Image and result rows, and thinking rows short enough to arrive whole, are published
  as a single event with `op: 'close'` (there is no separate `open`). Action, question and stage rows open and close.
  Image and result rows produced by a tool call carry that action's `rowId` as `parentRowId`.
- **Organization.** Routes without an id in the path (`uploads`, `projects`, `artifacts`) act in the caller's first
  organization, or in `?orgId=` when given. Anything owned by another organization answers 404.
- **Uploads.** `PUT /uploads/:id/:index` must carry exactly `chunkSize` bytes (the last chunk: the remainder);
  sending an index again replaces it. `complete` answers 409 naming the missing chunks, and returns the existing
  project when the same bytes were uploaded to the organization before.
- **Resume.** `POST /jobs/:id/resume` resumes a paused job and retries a failed or cancelled one from its stored
  conversation; on a running job it answers 409.
- **Trace stream.** After the last row of a finished job the stream sends `event: end` and closes. With the admin
  setting `VIDEO_TRACE_STEPS_ONLY=1`, thinking rows and deltas are left out; sequence numbers keep their values.
- **Translate.** `voiceover: true` or `lipSync: true` answers 422 until those stages exist; nothing is skipped
  silently.
- **Export.** 409 before a render exists, 410 if the rendered file is no longer on the server.

## 4. Editing a finished version

Editing starts when a version's job has ended; while the agent is still working these routes answer 409. Every
route needs write permission and answers 404 to another organization. Types are in `packages/shared/src/video.ts`
(`VideoEditedBlock`, `VideoBlockPatch`, `VideoEditEntry`, `VideoEditResult`, `VideoRenderResult`, `VideoProposal`).

### Worker tools (deterministic)

| Tool | Params | Result |
| --- | --- | --- |
| `set_block_text` | `{block_id, lines, script?}` | `{block, fit, warnings}` + before/after |
| `set_block_style` | `{block_id, size_scale?, weight?, color?, dx?, dy?, pitch?}` | same. Size is held to 0.9–1.1; `weight: 0`, `color: ''`, `pitch: 0` go back to the measured value |
| `set_block_timing` | `{block_id, in_shift, out_shift}` frames | same, no picture. `out_shift` may only be ≤ 0; any departure from the source is listed in `warnings` |
| `delete_block` / `restore_block` | `{block_id}` | leave the source text untouched / translate it again |
| `revert_block` | `{block_id}` | back to the baseline |
| `snapshot_baseline` | | records every block's present state as "the agent's version" (the job calls it when the agent finishes) |
| `get_state` | | `{blocks}`: every block with `target`, `fit`, `style`, `timing`, `deleted`, `used` (size, weight and pitch actually drawn), `on_screen` (first and last frame the text is inside the picture), `baseline`, `edited` |

Grouping, wording, style, timing and deletions are written to the session's folder and restored after a restart; the
rest is measured from the video again. A change to one block changes the cache key of only the segments that block
is drawn in.

### API

| Call | Body | Returns |
| --- | --- | --- |
| `GET /api/video/versions/:id` | | `VideoVersion` with edited blocks, `pending` (changes since the last render), `canUndo`, `canRedo`, `proposals`, `source` |
| `PATCH /api/video/versions/:id/blocks/:blockId` | `VideoBlockPatch` `{lines?, script?, style?, timing?, deleted?}` | `VideoEditResult` `{block, warnings, image?, entry?, pending, canUndo, canRedo}`. No `entry` when nothing changed |
| `POST .../blocks/:blockId/revert` | | `VideoEditResult`: back to the agent's version, as one new change |
| `GET .../history` | | `{entries: VideoEditEntry[], pending, canUndo, canRedo}`. Each entry: `n`, `at`, `by` (`user` or `ai`), `blockId`, `what`, `where`, `why?`, `undone` |
| `POST .../undo` \| `.../redo` | | `{entry, block, …counts}`; 409 when there is nothing to undo or redo. A new change after an undo drops what was undone |
| `POST .../history/:n/revert` | | `VideoEditResult`: takes back what change `n` changed and nothing else, as a new change |
| `POST .../render` | | `VideoRenderResult` `{segments, reEncoded: number[], seconds, qa, sizeBytes, …counts}`. Runs the worker's `export` then `qa_suite`; `reEncoded` is measured against the previous render's segment hashes |
| `POST .../instruct` | `{instruction, blockIds?, frameRange?, box?, frame?}` | 202 `VideoProposal` with `status: 'thinking'`. Scope: the named blocks, else those under `box` on `frame`, else those on screen in `frameRange`, else every translated block |
| `GET .../proposals` | | `{proposals}` |
| `POST .../proposals/:pid/accept` \| `reject` | | `{proposal, results: VideoEditResult[], …counts}`. Accepting applies the edits as changes `by: 'ai'` with the explanation as `why`; it does not render |
| `POST .../proposals/:pid/refine` | `{instruction}` | 202 a new `VideoProposal` with `parentId`; the old one is closed. Also how a `question` is answered |
| `GET /api/video/estimate` | | `{samples, secondsPerVideoMinute?}` from finished jobs in the workspace; nothing invented when there are none |

A proposal is made by a short model turn with three tools (`crop_zoom`, `propose_edits`, `ask_user`). Its edits are
tried on the worker to take the before/after picture and check the fit, then put back: nothing changes until it is
accepted. Its steps are published to the job's trace under one parent row, `fix-<proposal id>` (`traceRowId`).

`VideoStage` carries `activeMs` (time spent in the stage over every visit) and, while running, `resumedAt`; the first
`startedAt` and last `endedAt` of a stage overlap other stages and are not a duration.

## 5. Voice dubbing

Speech work is done by a voice provider (ElevenLabs) called from the API; the worker never sees the provider key and the provider never sees the picture. Audio moves between them through the API.

### Worker

| Call | Body | Returns |
| --- | --- | --- |
| `GET /sources/:sid/voice/audio?session=&which=source\|voice\|dub` | | WAV (the source track, the isolated voice, the finished dub); 404 when there is none |
| `PUT /sources/:sid/voice/upload?session=&kind=voice_stem\|phrase&name=` | raw audio in any common format | `{kind, name, seconds}`; 422 when it cannot be read |
| `GET /sources/:sid/export?session=&audio=auto\|original\|dub` | | the rendered file; `auto` is the dubbed file when one exists |

Worker tools (same `POST /sources/:sid/tools/:tool` call): `voice_state`, `voice_separate` (needs the `voice_stem` upload), `voice_set_phrases` `{phrases: [{id?, speaker, start_s, end_s, text}]}`, `voice_set_voice` `{speaker, voice_id, voice_name?}`, `voice_fit_phrases` `{items: [{id, spoken, display, voice_id?, anchor?: {block_id, mention}}]}` (needs each `phrase` upload), `voice_mix` `{mode: replace|second_track}`, `voice_backcheck` `{terms: [{phrase_id, term, found, heard_at_s?}], transcript}`. `qa_suite` adds the voice checks when a dub exists.

### Agent tools

`separate_stems`, `transcribe`, `list_voices`, `set_voice`, `dub_phrases`, `mix_and_mux`, `back_transcribe`. They are refused with a plain message when the job's `voiceover` option is off, and fail with "No ElevenLabs key is configured. Add one in Settings." when there is no key.

### API

| Call | Body | Returns |
| --- | --- | --- |
| `GET /api/settings/elevenlabs` | | `{elevenlabs: {configured, source: user\|shared\|none, masked?, defaultVoice?, voices, checkedAt?, problem?}}` |
| `PUT /api/settings/elevenlabs` | `{key}` | the status; 400 when ElevenLabs refuses the key |
| `POST /api/settings/elevenlabs/verify` | | the status, with `problem` when the key in use is refused |
| `DELETE /api/settings/elevenlabs` | | the status (voice choices are kept) |
| `PUT /api/settings/elevenlabs/voices` | `{defaultVoice?, voices?: {<language>: <voice id>}}` | the status |
| `GET /api/video/voices?search=&gender=&language=` | | `{configured, voices: LibraryVoice[], total, error?}` |
| `GET /api/video/versions/:id/voice` | | `{voiceover, phrases, speakers, dubbed, separated, mode, mix, usage}` |
| `PATCH /api/video/versions/:id/voice/phrases/:phraseId` | `{spoken, display?}` | the voice state; only that phrase is spoken again, then the dub is mixed and checked |
| `PUT /api/video/versions/:id/voice/speakers/:speaker` | `{voiceId, voiceName?}` | the voice state; that speaker's phrases are spoken again |
| `POST /api/video/versions/:id/voice/mode` | `{mode: replace\|second_track}` | the voice state |
| `GET /api/video/versions/:id/audio?which=source\|dub` | | WAV |
| `GET /api/video/versions/:id/export?audio=original\|dub` | | the file with that audio (default: the dub when there is one) |

`TranslateOptions` gains `voiceId?` and `voiceName?`. `voiceover: true` needs an audio track (422 without) and a voice key (409 without); `lipSync: true` is still 422.

### Changes of 2026-10-08 (dubbing fixes)

- `voice_separate` / `separate_stems` result: `background_kind` is `kept` (music, effects or ambience found and kept), `room_tone` (only room noise: generated tone at the same level is used) or `silence` (nothing kept); `residual_voice_db`, `method`, `gap_level_db`, `speech_level_db`, `aligned_by_ms`, `what_was_kept`. With `room_tone` or `silence` none of the source track is in the dub.
- `voice_set_phrases` / `transcribe` result: phrases of one speaker that overlap or are under 150 ms apart are merged into one unit (`merged_from` lists the original ids; `merged` lists merged units); `speaker_voices` gives `{pitch_hz, likely_register: low|mid|high|unknown}` per speaker.
- `voice_fit_phrases` / `dub_phrases` rows: `overlaps_next_by_s`; a phrase that does not fit at 1.10x is never cut and its `advice` says by how much to shorten the wording.
- `voice_mix` / `mix_and_mux` result: `residual_voice_db` (source voice under the new voice, dB relative to the new voice; -120 means none), `background_kind`; rows carry `overlaps_next_by_s`, never `cut_short: true`.
- `voice_backcheck` accepts `background_words` (words heard when a kept background is transcribed in the source language).
- New worker tool `voice_peaks {track: source|voice|background|dub, points?: 16..8000}` returns `{track, available, sample_rate, points, duration_s, samples_per_point, min: number[], max: number[]}`.
- New API route `GET /api/video/versions/:id/voice/peaks?track=&points=` returns that object, for drawing the voice lane's waveform.
- Worker audio: `GET /sources/:sid/voice/audio?which=background` serves the background; `PUT /sources/:sid/voice/upload?kind=background_stem&name=` stores a non-vocal stem from the provider (several are summed).
- `qa_suite` adds `voice_residual` (story I2.2, limit -30 dB), `voice_overlap` (a phrase running into the next) and, when a background is kept and transcribed, `voice_background_words`.


## Additions: delete, estimate and cost, captions

| Route | What it does |
| --- | --- |
| `GET /api/video/projects` | Now includes `versions` (`id, language, status, jobId, number`, and `error` when the job failed). |
| `DELETE /api/video/projects/:id` | Cancels live jobs, removes the project, its versions, traces, agent turns, artifact records, edit log, upload rows and the stored upload. The worker is asked to drop the source only when no other project has the same checksum. Returns `{ ok, versions, cancelled, sourceDropped }`. Audit: `video.project.delete`. |
| `DELETE /api/video/versions/:id` | The same for one translation; the worker drops that session. Returns `{ ok, cancelled }`. Audit: `video.version.delete`. |
| `GET /api/video/estimate?projectId=&languages=&onScreenText=&voiceover=&captions=` | `{ rows, total, minutes, languages, prices, measuredJobs, note }`. Each row: `id` (`onscreen`, `voiceover`, `captions`), `seconds`, `tokensRead`, `tokensWritten`, `voiceCharacters`, `costUsd`, `basis` (`measured` from this workspace's finished jobs, `typical`, or `rough`). Prices live in `apps/api/src/video/cost.ts`. |
| `GET /api/video/versions/:id/cost` | What the job used, same row shape, from stored turn usage, stage times and voice usage. |
| `GET` / `PUT /api/video/versions/:id/captions` | `{ enabled, cues, addTrack, warnings, limits }`; `PUT` takes `{ cues?, addTrack? }` and renders nothing. |
| `GET /api/video/versions/:id/captions.srt` and `.vtt` | The subtitle files. |
| `GET /api/video/versions/:id/export?subtitles=1` | The export with the captions as a soft `mov_text` track; video and audio packets are copied unchanged. |

Agent tool: `set_captions { cues: [{ start_s, end_s, text, source? }] }` (not strict). With captions on and voiceover off, `transcribe` is the one voice tool allowed.

Worker: `PUT /sources/{sid}/captions`, `GET /sources/{sid}/export?subs=1`, `DELETE /sources/{sid}/sessions/{name}`, `DELETE /sources/{sid}`.
