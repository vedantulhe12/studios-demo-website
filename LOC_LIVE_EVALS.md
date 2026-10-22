# Localization live evals

Run 2026-09-23 23:58 UTC with `node --env-file=.env --import tsx apps/api/test/live/loc-live-evals.ts google/gemini-3.7-flash anthropic/claude-opus-5.5 openai/gpt-6-luna`. Opt-in: real OpenRouter and ElevenLabs calls, keys from the gitignored `.env`, no database.

## Tool choice on the chat scenarios

The loc-lead skill, a project brief and the real tool schemas go to the model; each scenario scores the first tool call (and the next one after a canned result where the scenario needs it) against what the skill asks for. The same scenarios run offline, scripted, in `apps/web/e2e/zz-loc-journey-chat.spec.ts`.

### google/gemini-3.7-flash — 10/10

| Scenario | Tools called | Result | Time |
|---|---|---|---|
| capture-ambiguous-person | loc_team → ask_user | pass | 5.4 s |
| capture-resolved-person | loc_team → loc_capture_pair | pass | 5.8 s |
| four-pictures | ask_user | pass | 2.7 s |
| vague-request | ask_user | pass | 2.4 s |
| unknown-person | loc_team → ask_user | pass | 5.2 s |
| missing-language | ask_user | pass | 3.3 s |
| vendor-review | loc_stage_target | pass | 2.6 s |
| feedback | loc_task_status | pass | 2.8 s |
| glossary-import | loc_termbase_import | pass | 2.5 s |
| retranslate-rejected | loc_list_segments | pass | 2.6 s |

### anthropic/claude-opus-5.5 — 10/10

| Scenario | Tools called | Result | Time |
|---|---|---|---|
| capture-ambiguous-person | loc_team → ask_user | pass | 3.8 s |
| capture-resolved-person | loc_team → loc_capture_pair | pass | 3.5 s |
| four-pictures | ask_user | pass | 2.0 s |
| vague-request | ask_user | pass | 1.9 s |
| unknown-person | loc_team → ask_user | pass | 3.8 s |
| missing-language | ask_user | pass | 1.7 s |
| vendor-review | loc_stage_target | pass | 1.8 s |
| feedback | loc_team → loc_task_status | pass | 3.6 s |
| glossary-import | loc_termbase_import | pass | 1.7 s |
| retranslate-rejected | loc_list_segments | pass | 1.7 s |

### openai/gpt-6-luna — 8/10

| Scenario | Tools called | Result | Time |
|---|---|---|---|
| capture-ambiguous-person | loc_team → ask_user | pass | 2.6 s |
| capture-resolved-person | loc_team → ask_user | fail — expected loc_capture_pair | 2.9 s |
| four-pictures | ask_user | pass | 1.2 s |
| vague-request | ask_user | pass | 1.1 s |
| unknown-person | loc_team → ask_user | pass | 2.2 s |
| missing-language | ask_user | pass | 1.2 s |
| vendor-review | loc_stage_target | pass | 1.1 s |
| feedback | loc_team → ask_user | fail — expected loc_task_status or loc_review_feedback | 2.5 s |
| glossary-import | loc_termbase_import | pass | 1.7 s |
| retranslate-rejected | loc_list_segments | pass | 1.1 s |

## Speech (ElevenLabs)

- Scribe on fixtures/loc/media/clip-en.mp4 (a synthetic tone, no speech): 0 cues, heard “[dial tone]” — correct, nothing is said in it.
- Text-to-speech through `synthesize()` (English): 74 KB of audio/mpeg in 1.7 s, voice CwhRBWXzGAHq8TQ4Fs17 (the account’s own voice — the built-in one was refused on this plan)
- Scribe on that speech: 2 cues in 1.2 s, word recall 100 %
  - 0.10–1.02 s: Welcome to the course.
  - 1.58–4.26 s: Know your customer is required before you open an account.
- Text-to-speech (German dub line): 51 KB in 1.1 s, voice CwhRBWXzGAHq8TQ4Fs17

## What these evals changed

- **The skill.** The first run scored gemini-3.7-flash 8/10 and gpt-6-luna 8/10: after “no one called Priyanka” both went looking for screens instead of asking, and one model asked its question in plain text. `skills/loc-lead/SKILL.md` now says to always ask through `ask_user`, that the call after an unresolved name is `ask_user`, and that a page + languages + a resolvable person is enough to start. gemini-3.7-flash (the product default) went to 10/10; gpt-6-luna still asks one question too many when everything is specified (a smaller-model habit, not a missing rule).
- **Dubbing on free ElevenLabs plans.** Text-to-speech with the built-in voice answered 402 “Free users cannot use library voices via the API”, so every dub would have failed for such an account. `synthesize()` now falls back to one of the account's own voices when nobody chose one (and says what to do when a chosen voice is refused); covered offline by `loc-ux-repair.test.ts` against the fake service's free-plan mode.
- **The committed clip is a tone.** `fixtures/loc/media/clip-en.mp4` contains no speech, so the live round trip speaks a sentence through `synthesize()` and transcribes it back with Scribe instead.
