# Audit: what exists, what stays, what goes

Written for Phase 0 of the Video Translate rebuild (build brief `PROMPT.md`, §3.1), on branch `video-rebuild` off `dev`. Every path below was checked against the tree on 2026-10-07.

## Summary

- This repository is mainly **KaDeep Studios, a chat-first QA agent** (browser, mobile and desktop test automation). Localization is a second product kind inside it, and video is a thin slice of localization: about 2,100 lines of TypeScript on the API and one 816-line React component, six commits old.
- The account, storage, queue and realtime layers are sound and are kept.
- The video pipeline, the player and the on-screen editor cannot meet the brief's rules by construction and are replaced.
- Three things the brief assumes do not exist here: **billing**, **Postgres**, and **schema migrations**. Each is addressed below instead of being papered over.

## 1. What exists

### Stack and layout

npm workspaces (`apps/*`, `packages/*`), Node ≥ 20, TypeScript 5.7 run through `tsx` (the API has no build step).

| Path | What it is |
| --- | --- |
| `apps/api` | Express 4 + Socket.IO 4 API, agent loop and job workers. Entry `src/index.ts`, worker `src/worker.ts`. |
| `apps/web` | React 18.3 + Vite 5 single-page app. Tailwind v4 for utilities, hand-written CSS custom properties for the theme, no router library. |
| `apps/desktop` | Electron 33 shell around the web UI. |
| `apps/mobile` | Expo / React Native companion app. |
| `packages/shared` | The type contract (`src/index.ts`, `src/loc.ts`, `src/timeline.ts`). |
| `packages/agent` | OpenRouter tool-calling loop (`src/loop.ts`, `src/openrouter.ts`). |
| `packages/loc-formats` | Pure localization file filters (SCORM, XLIFF, Office, PDF, subtitles). |
| `packages/browser-driver`, `device-driver`, `desktop-driver`, `cli-agents` | QA drivers. Not touched by this work. |
| `services/memory-graph` | Python 3.12 FastAPI sidecar run with uv. The only Python in the repo before this work, and the pattern the new video worker follows. |
| `services/desktop-updates` | Small Node update-feed server. |
| `apps/briefing`, `apps/video`, `superadmin/`, `plugins/`, `t3code/` | Leftovers, marketing outputs, or a third-party checkout. No product code; ignored by this work. |

Tests: Node's built-in runner for the API (`apps/api/test/`), Playwright for end-to-end (`apps/web/e2e/`, two configs). CI is `.github/workflows/tests.yml`.

### Services and deployment

- Railway. `railway.web.toml` and `railway.memory-graph.toml` are in the repo; the API and worker services are configured in Railway itself (described in `docs/SCALING.md`).
- `docker-compose.yml` runs the API and web only. `apps/api/Dockerfile` installs ffmpeg, Noto fonts and Playwright engines.
- Nothing in this work deploys, and nothing connects to the production database. Development uses the local JSON store under `./data`.

### Data model

- **MongoDB** (driver 7) behind a custom `Store<T>` abstraction: `apps/api/src/db/store.ts` (`FileStore` and `MongoStore`), collections declared in `apps/api/src/db/models.ts`, indexes in `apps/api/src/db/indexes.ts`.
- When `TESTSTUDIOS_MONGODB_URI` is empty the store falls back to JSON files under `./data`.
- **There is no ORM and there are no schema migrations.** Collections are created on first write.

### Auth, organisations, roles

- Hand-rolled and complete: `apps/api/src/db/auth.ts` (HS256 JWT on `node:crypto`, scrypt passwords, rotating refresh tokens, per-device sessions), routes in `apps/api/src/routes/auth.ts`.
- Organisations, memberships, project memberships, teams, custom roles, invites and audit events: `apps/api/src/routes/orgs.ts`, `apps/api/src/db/capabilities.ts`.
- Capability checks such as `loc:translate` guard project routes.

### Billing

**None exists.** There is no payment vendor anywhere in the repo. What exists is a usage ledger and cost estimates (`apps/api/src/services/pricing.ts`, the `usage` collection, budgets). The brief expects billing to be kept; there is nothing to keep. Stories that mention credits, plans or a free tier (C5.1 "stops billing", N4, L4) are built against the usage ledger, and the missing payment layer is on the list of asks in `docs/PLAN.md`.

### Storage

- S3-compatible object store with multipart upload and Range reads, falling back to local disk: `apps/api/src/services/object-store.ts`. The localization wrapper is `apps/api/src/loc/storage.ts`.
- Chunked, resumable upload sessions already exist in `apps/api/src/routes/loc.ts`.

### Queue and workers

- A durable queue in Mongo: `apps/api/src/jobs/queue.ts` (atomic lease claim, heartbeat, checkpoint, pause, resume, cancel, retry with backoff, a capped job log).
- Workers: `apps/api/src/jobs/worker.ts` and `apps/api/src/jobs/loc-worker.ts`.
- It checkpoints by a case index, not by a content-hashed stage graph, and has no notion of a Python or GPU worker.

### Realtime

- Socket.IO is the main channel (`apps/api/src/index.ts`), with a typed event bus in `apps/api/src/services/events.ts`. Redis fan-out is optional.
- Server-sent events carry chat turns; the web client reads them with a fetch reader (`apps/web/src/api.ts`, `apps/web/src/loc/locApi.tsx`).
- An agent trace UI already exists for chat: `apps/web/src/chat/Timeline.tsx` over the reducer in `packages/shared/src/timeline.ts` (collapsible thinking and tool groups). It has no image rows.
- A job activity log exists for localization: `apps/web/src/loc/ui/Activity.tsx`. Text lines only.

### Model access

- Every model call goes through OpenRouter by raw `fetch` (`packages/agent/src/openrouter.ts`), with per-user keys resolved in `apps/api/src/services/llm-keys.ts`.
- **The Anthropic SDK is not used and there is no `ANTHROPIC_API_KEY`.** The brief requires the Claude API directly.

### UI routes

- No router. `apps/web/src/App.tsx` switches on workspace tabs (`apps/web/src/workspaceTabs.ts`).
- Localization lives in `apps/web/src/loc/`. The on-screen studio opens from `apps/web/src/loc/WorkbenchView.tsx`.
- Styling: `apps/web/src/styles.css` (dark tokens, dark is the default) and `apps/web/src/loc/loc.css` (localization overlay with the orange accent `#FF6602`).

### The current video pipeline

All of it is Node shelling out to an ffmpeg binary, plus model calls through OpenRouter.

| File | Role |
| --- | --- |
| `apps/api/src/loc/media.ts` (1,416 lines) | One module for three different things: **image** localization (`ocrAsset`, `renderLocale`), **audio and subtitle** tracks (`transcribeAsset`, `writeSubtitleTrack`, `dubLocale`), and **video on-screen text** (`detectOnScreenText`, `readOnScreenArea`, `addOnScreenLines`, `fillOnScreenTranslations`, `redrawStillCard`, `paintCardGroup`, `paintOneLine`, `judgePaintedLine`, `muxLocale`, `burnSubtitles`). |
| `apps/api/src/loc/onscreen.ts` (705 lines) | The pixel work: a colour-distance ink mask filled with a median colour, text set with ffmpeg `drawtext`, overlays with piecewise-linear motion expressions. |
| `apps/api/src/loc/voices.ts` | ElevenLabs key storage and synthesis. |
| `apps/api/src/services/video.ts` | Finds an ffmpeg binary. Also used by QA run recordings (`apps/api/src/agent/runner.ts`, `apps/api/src/scheduler.ts`, `apps/api/src/routes/resources.ts`). |
| `apps/api/src/routes/loc.ts` | Routes `…/assets/:assetId/onscreen/read`, `…/onscreen/lines`, `…/track/:locale/:kind`, and the `onscreen` patch on segments. |
| `apps/api/src/agent/tools/loc.ts` | Agent tools `loc_media` and `loc_localize_image`. |
| `packages/shared/src/loc.ts` | `LocOnScreenStyle`, `LocMediaTrack`, `parseOnScreenSheet`. |
| `apps/web/src/loc/workbench/OnScreenStudio.tsx` (816 lines) | The studio: two `<video>` elements kept together by copying `currentTime`, Text / Animation / Timing tabs, box handles. "Apply" re-renders the whole video. |
| `apps/web/src/loc/ui/MediaPlayer.tsx`, `apps/web/src/loc/workbench/CueEditor.tsx` | A `<video controls>` wrapper and a subtitle cue editor. |
| `apps/api/test/loc-onscreen.test.ts`, `apps/api/test/loc-media.test.ts` | Their tests. |

## 2. What to keep, and why

| Keep | Why |
| --- | --- |
| Auth, organisations, roles, audit events | Complete, tested, and nothing in them is video-specific. Roles and the audit log are what stories O1–O3 need. |
| `Store<T>` on MongoDB | It works and every other feature depends on it. The brief suggests Postgres "if choosing fresh"; we are not choosing fresh, and moving databases would put the rest of the product at risk for no gain to video. New data goes in new collections. |
| Object store and chunked uploads | Multipart and Range reads are what large video needs. The content-addressed artifact store is added beside the existing key helpers. |
| The Mongo job queue | Lease, heartbeat, checkpoint, pause, cancel and "waiting for a human" already exist. Video becomes a new job kind on it; Node keeps the lease and calls the Python worker over HTTP. |
| Socket.IO bus and the SSE pattern | The trace is delivered by SSE with resume; Socket.IO is only a nudge. |
| The chat trace UI's styles and reducer pattern | The new trace list is written in the same style and reuses the `tl-*` classes. The reducer itself is not extended: it is persisted on chat messages and caps and trims its parts. |
| Dark theme tokens and the localization orange accent | The owner chose to keep the orange accent with a Vozo-style layout. |
| Glossary, termbase and translation memory (`apps/api/src/loc/glossary.ts`, `termbase.ts`, `tm.ts`) | Story B3 reuses them. |
| ElevenLabs integration (`apps/api/src/loc/voices.ts`) | Becomes one provider behind the voice adapter in Phase 3. |
| Image localization and the audio/subtitle paths of `media.ts` | They serve the document-localization product and are not part of the video rebuild. |
| `apps/api/src/services/video.ts` | QA run recordings depend on it. |
| CI skeleton, Playwright setup, Railway layout | Extended, not replaced. |

## 3. What to replace, and why

| Replace | Why it cannot be fixed in place |
| --- | --- |
| Video on-screen functions in `apps/api/src/loc/media.ts` and all of `apps/api/src/loc/onscreen.ts` | Every paint pass re-encodes the whole video with default `libx264` settings, so pixels outside the text change and timestamps and colour tags are not preserved (rules 2.1, 2.2). Boxes are percentages estimated by a model and timing comes from a 4 fps sample, so 0-frame onsets and ≤ 1 px motion are out of reach. Text is set by ffmpeg `drawtext`, which does not guarantee complex-script shaping. Full-frame title cards are redrawn by an image-generation model. |
| Dub muxing in `media.ts` (`muxLocale`) | Re-encodes audio to AAC and drops the original track (rule 2.3). |
| `OnScreenStudio.tsx`, `MediaPlayer.tsx` (for video), `CueEditor.tsx` (for video) | Two `<video>` tags aligned by `currentTime` cannot be frame-accurate, which the brief makes a hard requirement. Every edit re-renders the whole video, so the 60-second partial re-render is impossible. |
| Model access for the orchestrator | The brief needs the Claude API directly (summarised thinking for the trace, prompt caching, image inputs). OpenRouter stays for the rest of the app. |

Whole videos are also read into memory as Buffers today; there is no frame index, shot detection, tracking, background model, stem separation, diarisation or lip-sync to build on.

## 4. What to delete at the end

**Removed on 2026-10-08.** The owner confirmed this list and asked for the old stored results to be deleted as well. Git history keeps the code recoverable. What was done, against the list below:

- **Whole files deleted:** all three.
- **`apps/api/src/loc/media.ts`:** every function named below is gone, with the `mux` branch of `runAvJob` and the burn-in option of `writeSubtitleTrack`. The `frame` reading mode and the preferred-model argument of `readRegions` went with them (nothing else used them).
- **Also removed, not on the original list because they only existed to serve the above:** the `onscreen` and `mux` steps of package translate jobs (`apps/api/src/loc/package-jobs.ts`), `updateOnScreen` in `apps/api/src/loc/segments.ts`, the muxed-video branch of a video delivery (`apps/api/src/loc/deliveries.ts`), the swap of a muxed video into an exported package (`apps/api/src/loc/package-media.ts`), `onScreenSheet` beside `parseOnScreenSheet`, and the `track` and `style` fields of a segment's context.
- **Routes, agent tool, shared types, CSS, tests:** as listed. `loc_media` now offers transcribe, subtitles and dub, and tells the agent that words on a video's picture belong to Video Translate.
- **Entry points:** the Workbench's "On-screen" tab and the "Edit on-screen text" / "On-screen text" buttons are gone. A video (on its own or inside a package) has one button, "Translate this video", that copies the file into Video Translate through its normal upload and opens it there.
- **Kept:** `MediaPlayer.tsx` and `CueEditor.tsx`, which still play and edit audio, video cues and course media; they lost only the muxed "Final video" track and the old studio button.
- **One behaviour changed as a result:** a dubbed video inside a course package is no longer rewritten with the new voice on export. The package keeps the original video and gains the subtitle track; a video with a new voice or new on-screen words comes from Video Translate.
- **Stored data:** `scripts/purge-old-onscreen-data.mjs` deletes old on-screen segments and documents and unsets `media.onscreen` and the `:mux` tracks. It is a dry run unless `--apply` is passed, always writes a backup first, and refuses to run on production without an explicit flag. The muxed files themselves stay in object storage; the backup lists their keys.

The original list, for reference:

**Delete whole files**

- `apps/api/src/loc/onscreen.ts`
- `apps/api/test/loc-onscreen.test.ts`
- `apps/web/src/loc/workbench/OnScreenStudio.tsx`

**Remove from files that stay**

- `apps/api/src/loc/media.ts`: `isOnScreen`, `videoReaderModel`, `detectOnScreenText`, `storeOnScreenClips`, `readOnScreenArea`, `addOnScreenLines`, `fillOnScreenTranslations`, `bannerLines`, `cardGroups`, `toOverlays`, `redrawStillCard`, `paintCardGroup`, `paintOneLine`, `judgePaintedLine`, `flagUnpainted`, `muxLocale`, `burnSubtitles`, and the `mux` branch of `runAvJob`. Image localization, transcription, subtitle files and audio-only dubbing stay.
- `apps/api/src/routes/loc.ts`: the `onscreen/read` and `onscreen/lines` routes, the `mux` kind of `track/:locale/:kind`, and the `onscreen` branch of the segment patch.
- `apps/api/src/agent/tools/loc.ts`: the video steps of `loc_media` (the tool is pointed at the new job kind instead).
- `packages/shared/src/loc.ts`: `LocOnScreenStyle`, `parseOnScreenSheet`, the `onscreen` field on media records, once no stored project still needs them (see §5).
- `apps/web/src/loc/loc.css`: the `.os-*` block (it starts at `.os-studio`).
- `apps/web/src/loc/WorkbenchView.tsx`, `AssetsView.tsx`, `ui/PackageContents.tsx`, `ui/CoursePlayer.tsx`: the entry points that open the old studio for video assets; they route to the new surface.
- `apps/api/test/loc-media.test.ts` and `apps/api/test/fake-services.ts`: the on-screen and mux cases.

**Keep, on review:** `apps/web/src/loc/ui/MediaPlayer.tsx` and `apps/web/src/loc/workbench/CueEditor.tsx` are also used for audio assets and course media. They lose their video role, not their file, unless the final review finds no remaining caller.

**Not part of this work and not deleted by it:** `apps/briefing`, `apps/video`, `superadmin/`, `plugins/`, `t3code/`.

## 5. Existing customer projects made with the old pipeline

*Superseded on 2026-10-08: the owner chose to delete the old results instead of preserving them (see §4). The plan below is kept as it was written.*

Old on-screen work is stored as `loc_segments` tagged `onscreen` (text, box, timing, style) plus rendered tracks under `asset.media.tracks`.

1. **Count first.** Before any removal, a read-only script reports how many organisations, projects, assets and segments carry on-screen data and how many rendered tracks exist. It is run against staging by default and against production only by the human.
2. **Preserve, read-only.** Existing rendered tracks stay downloadable exactly as they are. Their segments stay readable. The old studio's editing is retired; the data is not.
3. **Offer a re-run.** Each such asset shows one action, "Re-run with the new pipeline". It creates a new Video Translate project from the same source file. The user's existing translations are carried over as suggested text for matching blocks; nothing else is trusted from the old data (its boxes and timings are estimates).
4. **Nothing is overwritten.** The new version sits beside the old track until the user chooses to replace it.
5. **Removal is the last step.** The old fields and code are dropped only after the count is zero or the human accepts what remains, and the drop is listed in the removal confirmation.

## 6. Data changes and reversibility

There are no migrations to make reversible, so the rule is implemented as:

- **Additive only.** New collections (`video_sources`, `video_analyses`, `video_versions`, `video_renders`, `video_stage_cache`, `video_agent_turns`, `video_trace_events`, `video_edit_log`, `video_consents`, `video_qa_reports`) and new indexes. No existing collection or document shape is changed.
- **A documented drop script** removes exactly those collections and the per-organisation artifact prefix, returning the database to its prior state.
- **Never against production.** Local JSON store and staging only (rule 2.10).

## 7. Development machine

- Apple M2, 16 GB, no discrete GPU. Node 22 and uv are present.
- **Homebrew, ffmpeg, ffprobe, tesseract and Docker are not installed.** The plan assumed `brew install ffmpeg tesseract`; that is not possible here without the owner installing Homebrew.
- The worker therefore uses the FFmpeg libraries bundled in the PyAV wheel (decode, probe, packet comparison and libx264 encode all work without a binary). The licence consequence of that bundle is recorded in `docs/MODELS.md`.
- The harness's independent OCR (Tesseract) is installed in CI. On this machine the tests that need it skip with a loud message until it is installed; they are never reported as passing.

## 8. Surprises worth knowing

- The repo's main product is unrelated to video; video is roughly 2 % of the code.
- `media.ts` mixes image, audio and video work, so removal has to be function by function, not file by file.
- `localization_pipeline_code/` and its `.zip` at the repo root hold the hand-entered and measured clip-G1 data that the copy under `handoff/reference/reference_code/` deliberately leaves out. They are now ignored by git and covered by `scripts/check-no-handoff-refs.mjs`.
- `ELEVENLABS_API_KEY`, `ELEVENLABS_BASE_URL`, `ELEVENLABS_VOICE_ID`, `FFMPEG_PATH` and `AWS_S3_FORCE_PATH_STYLE` are read by code but missing from `.env.example`.
- The CI step for API tests skips every file with `video` in its name. New API tests for this work are named `vt-*.test.ts` and have their own step.
- Test media fixtures in the repo are two-second colour clips, so nothing today exercises real video content.
