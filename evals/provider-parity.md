# Provider parity (plan §9.6)

Since wizard v2 (decision 27) Anthropic, OpenAI, Azure OpenAI and Google Gemini are live with the organization's own
key. This file records the accuracy comparison against OpenRouter on the same model family; a provider row moves from
"pending" to a dated result the first time the run is done with a real direct key.

| Provider | Direct models | OpenRouter models compared | Runs | Result | Since |
| --- | --- | --- | --- | --- | --- |
| Anthropic | `claude-sonnet-4-6` + `claude-haiku-4-5` | `anthropic/claude-sonnet-4.6` + `anthropic/claude-haiku-4.5` | — | pending — needs a direct key | — |
| OpenAI | `gpt-5` + `gpt-5-mini` | `openai/gpt-5` + `openai/gpt-5-mini` | — | pending — needs a direct key | — |
| Azure OpenAI | deployment of `gpt-5` | `openai/gpt-5` | — | pending — needs a deployment | — |
| Google Gemini | `gemini-3.7-pro` + `gemini-3.7-flash` | `google/gemini-3.7-pro` + `google/gemini-3.7-flash` | — | pending — needs a direct key | — |

## Recipe

1. Start the eval API on a temp file store (`DB_BACKEND=file`, a fresh `DATA_DIR`) with the local Browserless shim.
2. Baseline on OpenRouter with the same family pinned (`AI_MODEL_LEAD` / `AI_MODEL_STEP`), three runs:
   `npm run evals -- --stories 8,88,102,103,104,106,136,156,272 --json evals/results/parity-<provider>-openrouter-<n>.json`
3. The direct run, three times: `EVALS_PROVIDER=<provider> EVALS_PROVIDER_KEY=… npm run evals -- --stories 8,88,102,103,104,106,136,156,272 --json evals/results/parity-<provider>-direct-<n>.json`
4. Acceptance: total passes ≥ the baseline, no story loses more than one of three, zero errors, p50 latency ≤ 1.5× the
   baseline, token totals within ±25 % (cache deltas explained). Record the row above with the date and commit.

Automated proof without a key: `apps/api/test/llm-direct-turn.test.ts` runs a chat turn, a file summary, preflight and
OpenAI / Gemini calls against fake provider servers with no OpenRouter key anywhere, and asserts that a direct provider
never receives an OpenRouter model id.
