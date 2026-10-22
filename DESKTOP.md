# Desktop testing

## How to set up (short)

1. Install KaDeep on the Mac/PC you want to test.
2. Grant Accessibility + Screen Recording (Mac only).
3. Open **Desktop** → **Pair this computer**.
4. Leave KaDeep running.

Use **Worker setup on this computer** in the Desktop pane for the permission doctor. Pairing happens in the app — you do not need an API URL. Watch the stream from the phone app (**Watch live**) — viewing only.

Native macOS / Windows / Linux apps are tested through a **paired worker machine**: the KaDeep Studios desktop app
(or its headless `--worker` mode) runs on the computer that has the app installed, drives it through
[cua-driver](https://github.com/trycua/cua) (trycua/cua, MIT — background accessibility + screenshot automation)
and streams state, frames and evidence back to the API. Stories 691–814. Cloud desktop VMs (cua Fleets / Lume) are
**coming soon**; the local worker is the shipped path and the UI says so wherever a cloud option would appear (735).

## Pieces

| Layer | Where | Role |
| --- | --- | --- |
| Driver contract | `packages/desktop-driver` | `DesktopDriver` (mirrors `DeviceDriver`): apps / windows, `snapshot()` (AT-SPI / AX / UIA tree + screenshot), click / type / press / hotkey / set value / scroll / drag / menus / clipboard, `verify()`, recording, health and permissions. `CuaDesktopSession` speaks MCP over stdio to `cua-driver mcp`; `MockDesktopSession` is the fixture used by tests and offline evals. Every action reports its **method** (`ax` / `px` / `key` / `api`) and whether the driver could confirm the effect (704). |
| Machine extensions | `packages/desktop-driver/src/ext.ts` | Install (DMG / PKG / MSI / EXE / DEB / AppImage) with stages and no silent elevation; file stat / validate / compare / list; performance sampling with leak suspicion; environment report (displays, scaling, locale); crash logs; uninstall leftovers; app scripting oracle (AppleScript / JXA / PowerShell COM / `.psjs`) restricted to allowlisted apps (723–760). |
| Bounds | `CapabilityManifest` | App allowlist (bounded mode), blocked OS surfaces (terminals, system settings) unless permitted, file writes contained to the project workspace, secure-field masking (705–709). |
| Worker registry + RPC | `apps/api/src/services/workers.ts`, `desktop/namespace.ts`, `desktop/remote.ts` | Pairing codes → worker credential; `/workers` Socket.IO namespace; `worker:call` RPC with acks (through the Redis adapter, so the job worker process can drive a machine whose socket lives on another replica); frames → the user's devices; `RemoteDesktopDriver` fulfils the contract over RPC and writes evidence into the run. |
| Agent tools | `apps/api/src/agent/tools/desktop.ts` | `desktop_*` group: start / status / list apps & windows / launch / focus / state / screenshot / zoom / verify / wait / click / double / right / type / type_variable / press / hotkey / set_value / scroll / drag / menu / window_frame / clipboard / quit / hand_to_human / install / uninstall_check / file / perf / env / crash_logs / run_script / recording / stop. Post-action `expect` verification, evidence screenshot per action, destructive guard (`confirmed=true`). |
| Runner | `agent/runner.ts` | `flowSurface()` routes `platform: desktop` (or desktop-labelled flows on a project with desktop bounds) to the worker; per-step screenshots plus an MP4 trajectory recording pulled into `run.video`; `run.desktop` records machine, display, driver version and the method counts. |
| Desktop app | `apps/desktop/src/main/worker/` | `cua-host.ts` (binary resolution: env → bundled `extraResources` → user install → PATH; macOS embedded host so TCC grants attribute to KaDeep Studios), `session-host.ts` (one `CuaDesktopSession` per API session, RPC dispatcher, HUD state), `client.ts` (outbound socket, pairing, heartbeat with health/permissions, reconnect), `overlay.ts` (transparent always-on-top click-through HUD with a Stop button), global stop hotkey `Ctrl/Cmd+Shift+Esc`, tray status, `permissions.ts` (first-run wizard with the driver doctor matrix), `updater.ts` (electron-updater, generic feed, Check for Updates, install on quit), deep link `kadeepstudiospro://worker/pair?token=…&api=…`, `--worker` headless mode. |
| Web | `apps/web/src/DesktopLiveView.tsx` | The Desktop pane: frames at ~2 fps, HUD (state, action, method, thinking), take control / hand back, worker list with the disabled **Cloud desktop VM — coming soon** entry, pairing codes, project bounds. |
| Fixture | `fixtures/canvas/` | Canvas Lite: a layered-canvas editor that runs as an Electron desktop target (`npx electron fixtures/canvas/main.cjs`) or in the browser, with a `__state()` oracle and five seeded defects for the desktop benchmark (759). |
| Skills | `skills/desktop-*`, `uxp-oracle`, `spectrum-aware` | `desktop-qa-lead` umbrella → smoke, lifecycle, files, creative; the oracle pattern for Adobe scripting; Spectrum component awareness (flagged "needs Adobe entitlement" when the real app is absent). |

## Pairing a machine (696)

1. Install the DMG / EXE and sign in. On macOS grant Accessibility and Screen Recording if asked. On Windows those
   grants do not exist — the driver is inside the install; do not run a PowerShell installer.
2. Open the **Desktop** tab. Inside the desktop app, **Pair this computer** works for any project member. That
   button posts `POST /workers/pair/self`, whose token is **bound to this machine's hostname and OS, lives two
   minutes and carries no deep link** — it cannot be forwarded to another computer. Pairing the same hostname
   again reuses the existing worker. A stale saved pairing can be cleared from the setup wizard or the tray
   (**Forget pairing**), which also deletes the row on the API so the project stops showing a ghost.
3. To attach a second machine, a lead uses **Pair another** and pastes the fresh 15-minute code (or opens the
   `kadeepstudiospro://worker/pair?token=…&api=…` link) on that machine. Testers paste a lead's code under
   **Permissions…**.
4. **One machine, one project.** Pairing a computer that already belongs to another project *moves* it: the old
   project's row is deleted, its default worker is cleared, and the app reports "Moved from &lt;project&gt;".
5. Leave KaDeep Studios running. Heartbeats every 15 s keep it online; 90 s of silence marks it offline, and a
   disconnect marks it offline immediately — an offline row is never resurrected by a recent heartbeat, and the
   agent is never told "1 online" about a machine it cannot reach. Remove offline leftovers from the pane (leads
   only) if an earlier test created extra workers.

### What each pairing phase means (1.1.2)

| Phase | Tray / wizard | What it is |
| --- | --- | --- |
| `unpaired` | Not paired | No credential saved. |
| `pairing` | Pairing… | A code is being redeemed; the paste box is hidden so a second code is never minted. |
| `connecting` | Reconnecting… | Transport retry with an attempt count. Never red — a dropped websocket is not an error. |
| `online` / `busy` / `paused` | Online / Busy / Paused | A socket is in the worker room. |
| `offline` | Offline | Disconnected, in plain English ("Connection closed — reconnecting…"). |
| `error` | Error: … | **Terminal only**: credential rejected, code invalid or used, or a `TESTSTUDIOS_API_URL` override that this machine is not paired to. |

With `TESTSTUDIOS_API_URL` set, the credential from that API stays in memory and is never written into
`worker.json` — a dev session cannot repoint the machine's real pairing.

Headless secondary machine: `KaDeepStudios --worker` (no main window; tray + overlay only). Pause / resume and
**Stop all sessions** live in the tray.

## Permissions (694–695)

- **macOS**: Accessibility and Screen Recording are keyed to the app's code signature. The embedded host
  (`@trycua/cua-driver/embedded`) starts the driver as a child of KaDeep Studios so the dialogs name KaDeep Studios.
  Unsigned / ad-hoc builds must re-grant after an update until a Developer ID signature is added (documented limitation).
- **Windows**: UIA needs no consent. The driver should be bundled in the EXE; if it is missing the app downloads the
  pinned GitHub zip into userData. Do not run the cua.ai PowerShell installer (default execution policy blocks it).
  Installer tools report BLOCKED when an MSI/EXE demands admin rights instead of prompting.
- **Linux**: X11 + AT-SPI. The driver needs the accessibility session bus; `discoverLinuxSessionEnv()` finds it from a
  running `at-spi2-registryd` so headless X11 workers (CI VMs) get real trees. Wayland support is BETA in cua-driver.

The first-run wizard shows the driver's `doctor` matrix and re-checks until green.

## How the agent drives an app (698–704)

`desktop_start` → `desktop_launch_app` (allowlist enforced) → `desktop_state`: the accessibility tree with `[index]`
handles plus a screenshot → act on indices (`ax` route, works even when the window is in the background) or on
window-local `x,y` for canvases (`px`) → re-read → `desktop_verify` for every recorded verification. Background input
delivery is tried first; when the platform has no focus-free route the driver says so and the action is re-sent in
the foreground, which the result records. Stale windows (a dialog we dismissed) are re-resolved to the app's main
window automatically. `desktop_hand_to_human` pauses for anything only a person may do on the machine (713).

## Coding-agent CLIs and the desktop method (1.1.0)

A chat that runs on Codex, Claude Code, Cursor or OpenCode already executes on the paired machine with that CLI's own
shell, file and scripting tools. For desktop work the chat's **Desktop method** (composer, beside the model and
effort; `ModelSelection.desktopMethod`) decides how it reaches the GUI:

- `auto` (default) — the agent picks per task: its own tools for launching and quitting apps, ports / processes /
  files / logs, AppleScript / JXA / PowerShell + UI Automation and quick smoke checks; the recorded `desktop_*`
  session for pixel- or control-level GUI work, when a per-action method record matters, or after two native failures.
- `cua` — the recorded session only; the shell is for files and diagnostics.
- `native` — the recorded driver is off for the chat: `desktop_*` tools other than `desktop_status` are not served over
  MCP, the `cua-driver` shim stays off the CLI's PATH, and the agent works the machine directly.

The setting travels on `AgentTurnRequest.desktopMethod`; `cli-turn.ts` filters the MCP tool list
(`toolsForDesktopMethod`) and adds the method block to the standing instructions (`desktopMethodBlock`); the Electron
`CliAgentHost` drops `DRIVER_NOTE` and the PATH shim for `native`. The `desktop-native` skill routes from
`desktop-qa-lead` and describes the native route.

Evidence in native mode comes through `attach_desktop_evidence`: the CLI passes the screenshot's path (or a small
inline PNG) plus a caption and `ok`; the API fetches the file from the machine over `worker:call` (`agent.evidence` —
images only, during a running turn, 12 MB cap), writes it under the run's artifacts, records a `RunAction`
(`method: 'api'`) with the screenshot, and broadcasts a live frame. A native check that is not attached is not evidence.

## Evidence (703)

Per action: a window screenshot into the run's artifacts, the method, the driver's own sentence, and
`unverified: true` when the driver could not confirm the effect. Per run: an MP4 trajectory recording from the worker
(`desktop_recording`, automatic on flow runs) pulled into `run.video` and published to evidence storage like browser
recordings. Console entries are the driver's log lines. `run.desktop.methods` counts how the run was driven.

## Local mode

`DESKTOP_LOCAL=1` lets the API drive its own display with cua-driver (a tester running the API on their laptop, or the
dev VM's X11 session). `DESKTOP_DRIVER=mock` uses the fixture driver (tests, offline evals).

## Building

`node apps/desktop/scripts/fetch-cua-driver.mjs [all|<os-arch>…]` downloads the release binaries into
`apps/desktop/resources/cua-driver/<os-arch>/` (git-ignored); electron-builder ships them under `extraResources` so
the OS can execute them. The version is pinned in `apps/desktop/cua-driver.version`. CI fails if the fetched binary
is missing. On Windows the app also looks in the official Cua install folders and, if still missing, downloads the
pinned zip into userData — do not tell people to run `irm https://cua.ai/driver/install.ps1 | iex` (default
PowerShell execution policy blocks it). macOS / Linux still fall back to a user install (`~/.local/bin/cua-driver`,
`PATH`) and the official `install.sh`.

## Tests

- `packages/desktop-driver/test` — manifest, mock contract, Linux bus discovery; with `CUA_LIVE=1` the real driver
  launches an app, reads its tree, types, records, and drives the Canvas Lite fixture to its seeded undo and save-as
  defects.
- `apps/api/test/desktop.test.ts` — pairing / registry, the tool group over the mock (bounds, method logging,
  verification, masking, evidence), extensions, runner routing; with `CUA_LIVE=1 DESKTOP_LOCAL=1` the tools drive a
  real app on the API's display.
- `apps/web/e2e/desktop-pane.spec.ts` — pane, cloud coming-soon entry, pairing code, bounds, and the 1.1.2 status
  edges: a transient websocket error never paints red, a second click mints no second token, the wait window is the
  token lifetime, "Moved from …", tester-hidden manage actions, project switch mid-pairing.
- `apps/desktop/src/main/worker/connect-errors.test.ts` + `pairing-ui.test.ts` + `apps/web/src/desktopPairing.test.ts`
  (`npm run test:desktop`) — the phase table, error classification and the pane's status reducer.
- `apps/desktop/e2e/pairing.spec.ts` (`npm run test:electron`) — the real Electron main process against a seeded
  file-backed API on port 8098 with a throw-away `--user-data-dir`: a tester pairs this computer, the project sees it
  online with a clean name, **Forget pairing** removes the row, re-pairing moves the machine to the other project,
  and no `wk_` credential is written while `TESTSTUDIOS_API_URL` is set.
- Verified on the dev VM end to end: API ⇄ headless Electron worker ⇄ cua-driver ⇄ Mousepad (39-element tree, masked
  typing, verification, recording, env / perf extensions, live frames).

## Limitations stated in the product

- Cloud desktop VMs: coming soon (picker entry disabled, tools answer with the same sentence).
- Unsigned builds re-ask for macOS permissions after updates.
- The driver only force-terminates processes it launched itself; foreign apps are quit cooperatively or handed to the person at the machine.
- Element verification can come back `unknown` from the driver on some desktops; the tools then decide from the same tree the agent sees and say so. Unknown is never a pass.
- Windows validation of the worker needs a Windows machine or the GitHub Actions matrix (EXE build); this pass validated Linux (X11) end to end and typed the macOS / Windows paths.
