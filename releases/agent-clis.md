# Coding-agent CLIs on a paired desktop

Experimental work on `desktop-experimentation/agent-clis`. A KaDeep chat can run on **OpenRouter** (today's loop) or on a **coding-agent CLI installed on the user's own computer**: Codex, Claude Code, Cursor, or OpenCode.

The API still owns the chat, the prompt, KaDeep tools, runs, evidence, persistence, and the live stream. The desktop app owns the CLI process and the user's subscription. Teammates never drive someone else's machine.

---

## What you can do

1. Pair the KaDeep Studios desktop app to a project (Desktop tab), same as today.
2. Open **Settings → Providers**. The app reports which CLIs are installed, whether you are signed in, and which models each one lists.
3. In the composer, pick a provider and a model. New chats remember the last choice for that project.
4. Send a message. If you picked a CLI, the turn runs on your paired machine. KaDeep tools (browser, mobile, desktop, flows, runs, issues, memory, artifacts) still work — the CLI calls them over MCP.
5. Switch provider mid-chat. The new CLI gets a handoff (summary, recent messages, live desktop / last run) and continues the work.
6. Stop a turn from the UI or the desktop stop hotkey. Restart the desktop app and the same chat resumes the provider conversation when the CLI supports it.

OpenRouter chats are unchanged. A CLI turn does not need an OpenRouter key. If a key is present it still powers sub-agents, summaries, and memory curation.

---

## What is new

### Shared contracts (`packages/shared`)

- Provider kinds: `openrouter` plus `codex`, `claudeCode`, `cursor`, `opencode`.
- Snapshots a machine reports: installed, version, auth, models, status (`ready` / `warning` / `error` / `disabled`).
- `ModelSelection` on a chat and on each agent message (`provider`, `model`, optional `effort` and `workerId`).
- `cliSession` on a chat: which machine, which provider conversation cursor, last model.
- Worker RPCs: `agent.turn`, `agent.interrupt`, `agent.stop`, `providers.refresh`, `providers.update`, `providers.snapshot`.
- SSE event `provider` when a chat switches engines, plus `providerSwitch` stored on the agent message.

### New package: `@teststudios/cli-agents`

One driver per CLI. Probes never send a prompt and never spend usage. Sessions stream text, native tool activity, and usage back as a common event shape.

| Provider     | How it is driven                         | Probe                                         | Session                                                                 | Default in Settings |
| ------------ | ---------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------------- | ------------------- |
| Codex        | `codex app-server` (JSON-RPC stdio)      | initialize → account refresh → model list     | One app-server per chat; MCP via env bearer; full access                | On                  |
| Claude Code  | Claude Agent SDK + user's `claude`       | version, `auth status`, SDK init + usage ping | One streaming SDK query per chat; HTTP MCP; `bypassPermissions`         | On                  |
| Cursor       | `cursor-agent acp` (ACP over stdio)      | `about --format json`, then ACP model list    | One ACP process per chat; HTTP MCP; permission prompts auto-approved    | Off                 |
| OpenCode     | `opencode serve` + official SDK          | version, local server, configured providers   | One localhost server per chat (random password); MCP as a remote server | Off                 |

Also in this package:

- PATH repair so Finder / Start-menu launches still find CLIs that work in a terminal.
- Turn prompt composition: context blocks are sent once and only re-sent when their text changes.
- JSON-RPC peer, process spawn/timeout, snapshot helpers.

### API

- **`GET /api/projects/:id/providers`** — OpenRouter key status plus every local machine's last CLI report. Stale reports (older than 5 minutes) trigger a refresh while someone is looking.
- **`POST …/providers/refresh`** — re-check one provider or all, on a machine you paired.
- **`PATCH …/providers/:kind`** — turn a provider on/off or set a binary path. The path may only name that CLI (`codex`, `claude`, `cursor-agent`/`agent`, `opencode`).
- **`POST /mcp/agent`** — Streamable HTTP MCP. Bearer token `kmcp_…` scoped to one chat. `tools/list` and `tools/call` use the same `toolsForMode` + `AgentContext` as the OpenRouter loop (guards, redaction, run records, HUD, stop).
- Chat PATCH accepts `selection`. Deleting a chat closes its CLI session.
- Replica bus (`apps/api/src/services/bus.ts`): worker events and MCP tool calls reach the replica that owns the turn, even when the desktop socket or the MCP POST landed elsewhere. Without Redis, delivery is local (single process).

### Desktop app

- `ProviderRegistry` — `userData/providers.json`, probes at startup / refresh / setting change.
- `CliAgentHost` — one live CLI process per chat; idle close after 30 minutes; liveness ping every 5 seconds; interrupt on stop/pause/quit.
- Workspace: project's desktop workspace directory when it exists on this machine, otherwise `userData/agent-workspace/<projectId>`.
- `cua-driver` is put on the CLI's PATH (same driver as `desktop_*` tools). Raw driver calls are not recorded on the run and skip the app allowlist; standing instructions tell the model to prefer KaDeep tools.
- Main process bundles `@teststudios/cli-agents`. Claude and OpenCode SDKs stay in `node_modules` and load on first use (ESM-only).
- `TESTSTUDIOS_API_URL` points the window and the in-memory worker at another API without rewriting the machine's saved pairing.

### Web

- Composer picker is a **provider rail + models pane** (`ProviderModelPicker`): OpenRouter keeps the existing catalog; a CLI shows that CLI's models, effort levels, and a one-line readiness reason.
- **Settings → Providers** (`ProvidersSettings`): OpenRouter card, then each CLI on the viewer's machines (switch, binary path, install/login command, Refresh). Teammates see machines they did not pair, but cannot configure or run them.
- Bubbles show which provider wrote the message and a switch banner when the engine changed.
- Cursor and OpenCode marks (`assets/providers/`).

### Mobile

- Same selection model and picker data (`src/lib/providers.ts`).
- Composer and message bubbles understand CLI selections and provider logos.

### Other

- `.gitignore` now ignores all `graphify-out/` trees (nested maps plus machine-local files).
- Tests: `packages/cli-agents/test/cli-agents.test.ts`, `apps/api/test/cli-providers.test.ts`.

---

## How a CLI turn works

```
Web / phone
  │  POST chat message + ModelSelection
  ▼
API  runChatTurn
  │  OpenRouter → existing runLoop
  │  CLI kind  → runCliTurn
  │    • pick the viewer's own online machine (chosen worker, else default)
  │    • refuse teammates and offline / unready CLIs (one live re-check first)
  │    • issue or reuse a kmcp_ token for this chat
  │    • attach this turn's AgentContext to that token
  │    • worker:call  agent.turn
  ▼
Desktop worker  CliAgentHost
  │  reuse the live CLI, or start / resume it
  │  send only changed <kadeep_context> blocks + the user message
  │  stream worker:agent  { event | alive | done }
  ▼
CLI process  (Codex / Claude / Cursor / OpenCode)
  │  own tools: shell, files, web  → shown as "Codex: …" in the chat
  │  KaDeep tools → POST /mcp/agent  (Bearer token)
  ▼
API  MCP handler  (same replica, or forwarded over the bus)
     tool.handler(ctx)  →  recorded on the chat and the run
```

**Sessions.** One CLI process per chat. Later turns reuse it so the CLI keeps its own conversation. Switching provider or machine, or a new MCP token, closes the old process. After an API or desktop restart the API stores a provider-native cursor and the worker resumes that conversation when the CLI supports it.

**Handoff.** The first turn on a new CLI (no live session, no resume cursor) includes recent messages, summary, standing decisions, parked items, and whether a desktop session or run is live. The model is told to continue, not restart.

**Context.** Standing voice and tool rules go into the CLI's system / instructions once. Per-turn blocks (mode, project, memory, skills, handoff) are hashed; unchanged blocks are omitted so a long chat does not reprint the project every message.

**Stop and health.** Abort calls `agent.interrupt`. If the machine goes quiet for 45 seconds the API fails the turn. The worker pings `alive` every 5 seconds so a long MCP tool is not mistaken for a dead app. Tray pause, overlay stop, and quit close every CLI.

**Cost.** CLI usage tokens are recorded when the CLI reports them. Dollar cost for a CLI turn is treated as zero (the spend is the user's own subscription). OpenRouter pricing is unchanged.

---

## Ownership and safety

- Only the person who paired a machine can refresh, configure, or run its CLIs. A CLI has full shell and file access on that computer.
- Binary paths from Settings are allowlisted to that provider's own executable name.
- MCP tokens are random, hashed in Redis (24h TTL), and revoked when the chat's CLI session closes. The raw token never leaves the API → that worker.
- CLI failures are not classified as OpenRouter key problems.
- `spawn_agent` is withheld on a CLI turn when no OpenRouter key exists (sub-agents still run on OpenRouter).

---

## How to try it

1. Build and run the API and the desktop app from this branch.
2. Pair the desktop app to a project.
3. Install and sign in to at least one CLI (`codex login`, `claude auth login`, `cursor-agent login`, or `opencode auth login`).
4. Settings → Providers → Refresh. Codex and Claude Code are on by default; turn on Cursor or OpenCode if you want them.
5. Open a chat, pick that provider's model, send a message that needs a KaDeep tool (for example a browser snapshot) and one that needs the machine (for example a local file).

---

## Out of this commit

A local `t3code/` checkout (nested clone, ~6 GB) was used as reference while building this and is **not** on the branch.
