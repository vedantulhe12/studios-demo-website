# Models, libraries, fonts and vendors

One row per thing the Video Translate product uses or is considering, with its licence and whether commercial use is allowed (build brief §8, rule 2.7).

**How this was checked.** On 2026-10-07 each licence was read from a primary source: the package registry's metadata (PyPI, npm), the repository's licence file through the GitHub API or raw file, the model card on Hugging Face, or the bundled library's own build report. The link in each row is the source read. Where a source could not be reached the row says **unverified** and nothing is assumed. This is a record of what the licences say, not legal advice; the items in "Decisions needed" should be looked at by whoever signs off the product's licensing.

Versions are the ones installed in `services/video-worker` or current on the registry on that date.

## Decisions needed

These are the rows that are not a plain "yes".

1. **x264 and x265 are GPL.** The FFmpeg inside the PyAV wheel is built with `--enable-libx264 --enable-libx265`, and the wheel ships `libx264` and `libx265`. Both are GPL-2.0-or-later. Running them on our own servers to produce customers' files is a different situation from distributing the software, but it needs a deliberate decision: accept GPL terms for the worker, buy a commercial x264 licence, or encode with something else (a hardware encoder, OpenH264, or a commercially licensed build). Until decided, development uses the bundled encoder.
2. **H.264 and HEVC are patent-pooled** regardless of which encoder is used. The pool terms could not be fetched today (**unverified**); they need checking before commercial launch.
3. **Lip-sync has no clean open option yet.** Wav2Lip is non-commercial. MuseTalk's code and weights are stated as commercially usable but depend on other models that each carry their own licence. LatentSync's code is Apache-2.0 but its published weights are under an OpenRAIL++ licence with use restrictions. A hosted API behind the provider adapter is the likely route.
4. **Research models flagged in the brief are confirmed non-commercial:** ProPainter (S-Lab License 1.0) and CoTracker (CC-BY-NC 4.0). Neither is used.
5. **Demucs weights.** The code is MIT; the pretrained models were trained on MUSDB HQ plus extra songs, and the licence of the weights for commercial use is **unverified**.
6. **ElevenLabs' commercial-use terms** could not be fetched today (**unverified**). It is also a processor of customer audio and must be declared as one (rule 2.9).

## In use from Phase 0 and Phase 1

### Video, numerics, imaging

| Name and version | Licence | Commercial use | Where used | GPU / memory | Fallback |
| --- | --- | --- | --- | --- | --- |
| PyAV (`av`) 19.0.1 | BSD-3-Clause ([PyPI](https://pypi.org/pypi/av/json), [LICENSE](https://github.com/PyAV-Org/PyAV/blob/master/LICENSE.txt)) | Yes, for PyAV itself | Decode, probe, packet comparison, encode, mux, in the worker and the harness | CPU | A system FFmpeg binary |
| FFmpeg libraries bundled in the PyAV wheel (libavcodec 63.1.102) | The build reports "LGPL version 3 or later" (`--enable-version3`), read from the installed wheel's own `library_meta`. **It links libx264 and libx265, which are GPL** (see next rows) | **Needs a decision** (item 1) | Everything PyAV does | CPU | An LGPL-only FFmpeg build without x264/x265 |
| x264 (libx264.165, bundled) | GPL-2.0 ([COPYING](https://github.com/mirror/x264/blob/master/COPYING)). Availability and terms of a commercial licence: **unverified** (x264.org returned 403) | **GPL: flag.** Needs a decision (item 1) | H.264 export and the review proxy | CPU | Hardware encoder (VideoToolbox is in the same build), OpenH264, or a commercial licence |
| x265 (libx265.217, bundled) | GPL-2.0-or-later per the x265 project; **unverified today** (not fetched) | **GPL: flag** | HEVC export (Phase 4) | CPU | Hardware HEVC encoder |
| H.264 / HEVC patent pools | Pool terms **unverified** (the Via LA page returned 404) | **Needs a decision** (item 2) | Any H.264 or HEVC export | – | AV1 or VP9 export where the source allows |
| NumPy 2.5.3 | BSD-3-Clause AND 0BSD AND MIT AND Zlib AND CC0-1.0 ([PyPI](https://pypi.org/pypi/numpy/json)) | Yes | All array work | CPU; frames are memory-mapped | – |
| OpenCV (`opencv-python-headless`) 5.0.0.93 | Apache-2.0 ([PyPI](https://pypi.org/pypi/opencv-python-headless/json), [LICENSE](https://github.com/opencv/opencv/blob/5.x/LICENSE)). The wheel bundles its own FFmpeg libraries; their build flags were not inspected (**unverified**) | Yes for OpenCV | Warps, morphology, registration, feature matching | CPU | SciPy and NumPy for the same maths |
| SciPy 1.18.1 | BSD ([PyPI](https://pypi.org/pypi/scipy/json)) | Yes | Filters, fitting, signal processing | CPU | – |
| Pillow 12.3.0 | MIT-CMU ([PyPI](https://pypi.org/pypi/pillow/json)) | Yes | Image encode for artifacts and thumbnails | CPU | OpenCV encoders |
| pyloudnorm 0.2.0 | MIT ([PyPI](https://pypi.org/pypi/pyloudnorm/json), [LICENSE](https://github.com/csteinmetz1/pyloudnorm/blob/master/LICENSE)) | Yes | Loudness measurement in the harness and the mixer | CPU | FFmpeg's `ebur128` filter |
| FastAPI 0.142.2 / Uvicorn 0.54.0 / Pydantic 2.13.5 | MIT / BSD-3-Clause / MIT ([FastAPI](https://pypi.org/pypi/fastapi/json), [Uvicorn](https://pypi.org/pypi/uvicorn/json), [Pydantic](https://pypi.org/pypi/pydantic/json)) | Yes | The worker's HTTP interface | CPU | – |
| pytest 9.1.1 | MIT ([PyPI](https://pypi.org/pypi/pytest/json)) | Yes (test only) | Harness and worker tests | – | – |

### Text shaping and fonts

| Name and version | Licence | Commercial use | Where used | GPU / memory | Fallback |
| --- | --- | --- | --- | --- | --- |
| uharfbuzz 0.56.3 | Apache-2.0 ([PyPI](https://pypi.org/pypi/uharfbuzz/json), [LICENSE](https://github.com/harfbuzz/uharfbuzz/blob/main/LICENSE)) | Yes | Complex-script shaping | CPU | Pillow with libraqm |
| HarfBuzz (inside uharfbuzz) | "Old MIT" ([COPYING](https://github.com/harfbuzz/harfbuzz/blob/main/COPYING)) | Yes | Shaping engine | CPU | – |
| freetype-py 2.5.1 | BSD (new/revised) ([LICENSE](https://github.com/rougier/freetype-py/blob/master/LICENSE.txt)) | Yes | Glyph rasterising | CPU | Pillow's FreeType |
| FreeType (inside freetype-py) | FreeType License (FTL) or GPLv2, at the user's choice ([freetype.org](https://freetype.org/license.html)) | Yes, under the FTL (requires a credit in documentation) | Glyph rasterising | CPU | – |
| Noto Sans | SIL OFL 1.1 ([OFL](https://github.com/notofonts/latin-greek-cyrillic/blob/main/OFL.txt)) | Yes, including embedding in rendered video | Latin target text; default | – | Any OFL Latin family |
| Noto Sans Devanagari | SIL OFL 1.1 ([OFL](https://github.com/notofonts/devanagari/blob/main/OFL.txt)) | Yes | Hindi target text; default | – | Hind, Mukta, Poppins |
| Poppins | SIL OFL 1.1 ([OFL](https://github.com/itfoundry/Poppins/blob/master/OFL.txt)) | Yes | Font matching (the clip-G1 reference used it) | – | Noto Sans Devanagari |
| Hind | SIL OFL 1.1, read from the Google Fonts copy ([OFL](https://github.com/google/fonts/blob/main/ofl/hind/OFL.txt)). The foundry's own repository has no licence file | Yes, using the Google Fonts distribution | Font matching alternative | – | Noto Sans Devanagari |
| Mukta | SIL OFL 1.1 ([OFL](https://github.com/EkType/Mukta/blob/master/OFL.txt)) | Yes | Font matching alternative | – | Noto Sans Devanagari |

Font files reach the server today through `@expo-google-fonts/*` npm packages (used by the old pipeline). The font files are the OFL families above; the packaging's own licence was not checked (**unverified**). The new worker will vendor the OFL files directly and record each file's hash.

User-uploaded fonts (story B4) are the user's responsibility and require their licence confirmation.

### Text detection and recognition

| Name and version | Licence | Commercial use | Where used | GPU / memory | Fallback |
| --- | --- | --- | --- | --- | --- |
| RapidOCR 3.9.2 | Apache-2.0 ([PyPI](https://pypi.org/pypi/rapidocr/json), [LICENSE](https://github.com/RapidAI/RapidOCR/blob/main/LICENSE)) | Yes | Pipeline text detection and OCR | CPU, a few hundred MB | Tesseract |
| PP-OCRv5 detection models (mobile and server) | Apache-2.0 ([model card](https://huggingface.co/PaddlePaddle/PP-OCRv5_mobile_det), [PaddleOCR LICENSE](https://github.com/PaddlePaddle/PaddleOCR/blob/main/LICENSE)) | Yes | Text detection | CPU | PP-OCRv4 detection |
| PP-OCRv5 Latin recognition model | Apache-2.0 ([model card](https://huggingface.co/PaddlePaddle/latin_PP-OCRv5_mobile_rec)) | Yes | Recognising Latin-script text | CPU | Tesseract `eng` |
| PP-OCRv5 Devanagari recognition model | Apache-2.0 ([model card](https://huggingface.co/PaddlePaddle/devanagari_PP-OCRv5_mobile_rec)) | Yes | Recognising Hindi source text and reading back Hindi output | CPU | Tesseract `hin` |
| RapidOCR's ONNX conversions of the above | Distributed by RapidOCR from ModelScope; listed in its [default models file](https://github.com/RapidAI/RapidOCR/blob/main/python/rapidocr/default_models.yaml). The conversions' own licence statement was not read (**unverified**); the upstream models are Apache-2.0 | Expected yes; confirm | As above | CPU | Convert the Apache-2.0 Paddle models ourselves |
| ONNX Runtime 1.30.0 | MIT ([PyPI](https://pypi.org/pypi/onnxruntime/json), [LICENSE](https://github.com/microsoft/onnxruntime/blob/main/LICENSE)) | Yes | Runs the OCR models | CPU (CoreML or CUDA optional) | – |
| Tesseract and tessdata (`eng`, `hin`) | Apache-2.0 ([tesseract](https://github.com/tesseract-ocr/tesseract/blob/main/LICENSE), [tessdata](https://github.com/tesseract-ocr/tessdata/blob/main/LICENSE)) | Yes | **Harness only**: the remnant scan uses an OCR the pipeline does not, so the check is independent | CPU | Skipped loudly where it is not installed; never reported as a pass |

### Front end and orchestrator

| Name and version | Licence | Commercial use | Where used | GPU / memory | Fallback |
| --- | --- | --- | --- | --- | --- |
| `@anthropic-ai/sdk` 0.131.0 | MIT ([npm](https://registry.npmjs.org/@anthropic-ai/sdk/latest), [LICENSE](https://github.com/anthropics/anthropic-sdk-typescript/blob/main/LICENSE)) | Yes | The Opus 5.5 agent loop | – | – |
| Claude Opus 5.5 (`claude-opus-5-5`), Anthropic API | Commercial API under Anthropic's terms (not fetched today: **unverified**). **Processor:** receives sampled frames, crops and text from customer videos | Yes, as a paid API | Planning, grouping, classification, translation, visual review, edit instructions | Hosted | None by design: the brief names this model. A release run is never stubbed |
| mp4box.js 2.4.1 | BSD-3-Clause ([npm](https://registry.npmjs.org/mp4box/latest), [LICENSE](https://github.com/gpac/mp4box.js/blob/main/LICENSE)) | Yes | Demuxing the review proxy in the browser | – | Server PNG frame endpoint |
| WebCodecs | Browser API | Yes | Decoding the review proxy | Client hardware | Server PNG frame endpoint |
| `@tanstack/react-virtual` 3.14.13 | MIT ([npm](https://registry.npmjs.org/@tanstack/react-virtual/latest), [LICENSE](https://github.com/TanStack/virtual/blob/main/LICENSE)) | Yes | The virtualised trace list | – | – |

## Phase 3 and later: audio

| Name and version | Licence | Commercial use | Where used | GPU / memory | Fallback |
| --- | --- | --- | --- | --- | --- |
| faster-whisper 1.2.1 (CTranslate2 4.8.2) | MIT / MIT ([faster-whisper](https://github.com/SYSTRAN/faster-whisper/blob/master/LICENSE), [PyPI](https://pypi.org/pypi/ctranslate2/json)) | Yes | Speech recognition with word timings; back-transcription in the harness | CPU works; GPU for speed. 2–4 GB for large models | A hosted speech-recognition API behind the adapter |
| Whisper weights (large-v3, large-v3-turbo) | MIT ([openai/whisper](https://github.com/openai/whisper/blob/main/LICENSE), [large-v3-turbo card](https://huggingface.co/openai/whisper-large-v3-turbo), [Systran conversion](https://huggingface.co/Systran/faster-whisper-large-v3)) | Yes | As above | As above | – |
| Silero VAD 6.2.3 | MIT ([LICENSE](https://github.com/snakers4/silero-vad/blob/master/LICENSE)) | Yes | Phrase boundaries | CPU | Energy-based segmentation |
| ElevenLabs (speech, voice cloning; `scribe_v1` transcription is used by the old pipeline) | Hosted vendor. Commercial-use terms **unverified** (help page returned 403). **Processor:** receives customer audio and, for cloning, voice samples | **Unverified**; believed to depend on the plan | Speech generation per speaker; consented voice clones | Hosted | Another provider behind the same adapter |
| pyannote.audio 4.0.7 | MIT for the code ([LICENSE](https://github.com/pyannote/pyannote-audio/blob/main/LICENSE)) | Yes for the code | Speaker diarisation (candidate) | GPU preferred | A hosted diarisation API |
| pyannote models: `speaker-diarization-3.1`, `segmentation-3.0` | MIT, **gated** (the user must accept conditions on the model page) ([card](https://huggingface.co/pyannote/speaker-diarization-3.1)) | Yes, after accepting the gate | As above | As above | – |
| pyannote model: `speaker-diarization-community-1` | CC-BY-4.0, **gated** ([card](https://huggingface.co/pyannote/speaker-diarization-community-1)) | Yes, with attribution | As above | As above | – |
| Demucs 4.1.0 | MIT for the code ([LICENSE](https://github.com/facebookresearch/demucs/blob/main/LICENSE)). Pretrained weights were trained on MUSDB HQ plus 800 songs ([README](https://github.com/adefossez/demucs/blob/main/README.md)); the weights' terms for commercial use are **unverified** | **Unverified for the weights** (item 5) | Voice / music / effects separation (candidate) | GPU preferred; several GB | A hosted separation API, or a model trained on commercially cleared data |

## Phase 3 and later: lip-sync, tracking, fill

| Name | Licence | Commercial use | Status |
| --- | --- | --- | --- |
| Wav2Lip | No licence file. The README says results are for research, academic and personal use only and "any form of commercial use is strictly prohibited" ([README](https://github.com/Rudrabha/Wav2Lip/blob/master/README.md)) | **No. Not used.** | Rejected |
| MuseTalk | MIT for the code; the README says the trained models are available for any purpose, even commercially, and that the other models it depends on (Whisper, a VAE, DWPose, S3FD and others) must each comply with their own licences ([LICENSE](https://github.com/TMElyralab/MuseTalk/blob/main/LICENSE), [README](https://github.com/TMElyralab/MuseTalk/blob/main/README.md)) | Stated yes; **each dependency's licence is unverified** | Candidate, pending a dependency-by-dependency check |
| LatentSync | Apache-2.0 for the code ([LICENSE](https://github.com/bytedance/LatentSync/blob/main/LICENSE)); published weights are under OpenRAIL++ ([card](https://huggingface.co/ByteDance/LatentSync-1.6)), which carries use restrictions | **Needs review** of the OpenRAIL++ terms | Candidate |
| Hosted lip-sync APIs | Vendor terms; none fetched (**unverified**). Each is a processor of customer video | Expected yes on paid plans; unverified | The likely route, behind the provider adapter. A comparison with costs is owed before the vendor ask is answered |
| A SyncNet-class sync scorer (story V6.5) and a face-embedding model (V6.6) | Not yet chosen; **unverified** | – | To be selected with a licence check; these are test instruments as well as product parts |
| ProPainter | S-Lab License 1.0: redistribution and use for non-commercial purposes only ([LICENSE](https://github.com/sczhou/ProPainter/blob/main/LICENSE)) | **No. Not used.** | Rejected. Temporal fill is built from measured motion instead |
| CoTracker | CC-BY-NC 4.0 ([LICENSE](https://github.com/facebookresearch/co-tracker/blob/main/LICENSE.md)) | **No. Not used.** | Rejected. Tracking uses OpenCV methods |
| SAM 2 (code and `sam2.1` weights) | Apache-2.0 ([LICENSE](https://github.com/facebookresearch/sam2/blob/main/LICENSE), [card](https://huggingface.co/facebook/sam2.1-hiera-large)) | Yes | Candidate for segmenting protected objects, hands and occluders in footage | GPU; several GB | Classical mattes, as used for clip-G1 |
| Generative fill model | Not chosen. Gated, labelled, last resort (story F5) | – | To be selected with a licence check |

## Processors of customer media (rule 2.9, story O1)

No customer media is sent to a third party that is not on this list and configured.

| Processor | Receives | Status |
| --- | --- | --- |
| Anthropic (Claude API) | Sampled frames, zoom crops, text, transcripts | Required from Phase 1; needs the owner's key |
| ElevenLabs | Audio for transcription or cloning; text for speech | Already wired in the old pipeline; to be declared in the product's processor list |
| Object storage (S3-compatible, currently a Railway bucket) | Source files, artifacts, renders | Existing |
| A lip-sync vendor | Face video and dubbed audio | Not chosen |
| A separation or diarisation vendor | Audio | Only if the open models are not used |

OpenRouter is used by the rest of the app, not by Video Translate.

## Could not be verified today

- x264 commercial licence availability and terms; x265's licence text.
- H.264 and HEVC patent pool terms.
- ElevenLabs commercial-use terms; Anthropic's API terms.
- Build flags of the FFmpeg bundled inside the OpenCV wheel.
- The licence statement on RapidOCR's ONNX model conversions.
- The licence of the `@expo-google-fonts/*` packaging.
- Demucs pretrained weights for commercial use.
- Every dependency of MuseTalk; LatentSync's OpenRAIL++ restrictions in detail; any hosted lip-sync vendor's terms.

## ElevenLabs as used by Video Translate's dubbing (added 2026-10-08)

ElevenLabs is a **processor that receives customer audio**: the source audio track of every video that is dubbed (for voice isolation and transcription), the finished dub (for back-transcription), and the translated text of every phrase (for speech). It does not receive the picture. It must be declared in the product's processor list before dubbing is offered to customers (brief rule 2.9). Its commercial-use terms are still **unverified** here.

Endpoints and field names were read from the API reference at elevenlabs.io/docs/api-reference on **2026-10-08** and are what `apps/api/src/video/voice/elevenlabs.ts` sends:

| Use | Call | Fields used |
| --- | --- | --- |
| Transcription with word times and speakers | `POST /v1/speech-to-text` (multipart) | `model_id` (`scribe_v2`, override with `VIDEO_ASR_MODEL`), `file`, `diarize`, `timestamps_granularity=word`, `tag_audio_events=false`, `language_code`; reply `language_code`, `text`, `words[]` with `text`, `type`, `start`, `end`, `speaker_id` |
| Voice isolation | `POST /v1/audio-isolation` (multipart) | `audio`; reply is the isolated voice as audio |
| Voice library | `GET /v2/voices` | `page_size`, `search`, `next_page_token`; reply `voices[]` with `voice_id`, `name`, `category`, `labels` (`gender`, `accent`, `age`, `language`, `use_case`), `preview_url`, `description`, `verified_languages[]`, plus `has_more`, `next_page_token` |
| Speech | `POST /v1/text-to-speech/{voice_id}?output_format=mp3_44100_128` | `text`, `model_id` (`eleven_multilingual_v2`, override with `VIDEO_TTS_MODEL`); reply is audio |

Not verified: the reference page for speech lists no response header for character usage, so the product counts the characters it sends and the seconds of audio it uploads itself; it does not claim a cost. A stem-separation endpoint (`POST /v1/music/stem-separation`, a ZIP of stems) also exists, but the reference does not name the files in the ZIP, so the background is instead rebuilt here from the original minus the isolated voice.

No call to ElevenLabs has been made by this code yet: there is no key in the environment. Every test runs against `apps/api/test/fake-elevenlabs.ts`, which is labelled as a fake.

### ElevenLabs stem separation (checked 2026-10-08)

`POST /v1/music/stem-separation` exists in the API reference: multipart field `file`, optional `stem_variation_id` and `output_format`, and it returns a ZIP with one audio file per stem. The reference page does not list the values of `stem_variation_id` or the names of the files in the ZIP. The adapter calls it with the default variation, only when the pipeline's own separation finds music or sound worth keeping, and treats every stem whose name does not look like the voice as background; the worker then uses the provider's background only if it measures less source voice in it than in its own. **This path has not been run against the live service** (the only real dub so far, clip-G1, has no music), so its ZIP layout, cost and plan requirements are unverified. The pipeline's own spectral-mask separation is the tested path.

