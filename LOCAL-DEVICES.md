# Local mobile devices (paired computer) beside Appetize

A project can run each mobile platform on the paired computer instead of Appetize: an Android emulator on Windows or
macOS, an iOS Simulator on macOS. Appetize bills per minute; exploratory runs and agent-written cases spend most of
their time thinking and waiting, so the lane that costs nothing per minute is the default we want for iteration, and
Appetize stays for everything local cannot honestly do.

This page is the decision record: what we found in the repo, what we compared, what we chose, what we refused to
build, and where the gaps are. The product doc for the Appetize lane is [MOBILE.md](MOBILE.md); the paired-machine
transport is [DESKTOP.md](DESKTOP.md).

## The setting

Settings → Devices chooses a home **per platform**: Android on Appetize or on this computer, iOS on Appetize or on
this computer (`project.mobile.runOn`). There is no automatic fallback. If the chosen computer is offline, or has no
runnable emulator / simulator, the run fails with the fix ("open KaDeep Studios on …", "create an AVD with …",
"switch Android to Appetize") so nobody spends Appetize minutes they did not choose. The same tab picks the machine,
a **Visible** or **Headless** window on that machine, and the AVD / simulator to use. The web shows a view-only live
stream; take-control stays an Appetize feature.

## What the repo already had

- `packages/device-driver` defines `DeviceDriver`, the `DeviceProvider` registry (tier `'local'` was reserved),
  `DeviceCapabilities` and `DeviceCapabilityError`. Appetize's `pinch()` throws "no multi-touch"; the tools turn that
  into INCONCLUSIVE. A local lane implements the same contract, so `tools/mobile.ts`, the runner, replay, suites and
  reports do not change.
- The paired-machine transport: `RemoteDesktopDriver` (`apps/api/src/desktop/remote.ts`) is one `worker:call` per
  method; the Electron `SessionHost` holds the real driver behind method allow-lists. The device lane mirrors both
  (`RemoteDeviceDriver`, `DeviceSessionHost`).
- Detection: the desktop app repairs PATH (GUI launches have no shell PATH), probes CLIs in `ProviderRegistry`, and
  ships snapshots on the heartbeat only when they change; the API sanitizes them. The device inventory rides the same
  heartbeat.
- The home switch: `project.browsers.target` and `openDesktopHome()` — an explicit choice with a clear error when
  the computer is offline. Same rule here.
- `hierarchy.ts` already parses uiautomator XML and WebDriverAgent `XCUIElementTypeApplication` XML.

## What each OS can host

| Host | Android emulator | iOS Simulator |
|---|---|---|
| Windows x64 | yes — x86_64 images with WHPX (AEHD is retired after Dec 2026, HAXM is gone) | never offered |
| Windows on ARM | not supported by Google's emulator — stays on Appetize | never offered |
| macOS Apple Silicon | yes — arm64-v8a images only | yes, with full Xcode |
| macOS Intel | yes — x86_64 images | yes, with full Xcode |
| Linux | Android works with the same code (KVM); not a tested target | never offered |

Images the host cannot run (an x86_64 image on Apple Silicon) are listed with the reason and never offered. A Mac with
only Command Line Tools selected but Xcode installed is handled: detection finds `/Applications/Xcode*.app` and runs
`xcrun` with `DEVELOPER_DIR` pointed at it, without touching the user's `xcode-select`. With no Xcode at all, iOS is
reported unavailable with the fix.

## What we compared

Android:

- **Emulator gRPC controller** (`-grpc <port> -grpc-use-token`): `sendTouch` takes several touches with identifiers
  mapped to kernel multi-touch slots — real pinch, two-finger rotate, N-finger paths; also `getScreenshot`,
  `setClipboard`, `setGps`, `setPhysicalModel`, `sendFingerprint`, `getStatus`. Ships inside the emulator; nothing is
  installed on the device. The proto is loaded from the user's own install (`emulator/lib/emulator_controller.proto`).
- **Emulator console `event send`** (multi-touch protocol B): documented by Google with a pinch example; chatty but
  works on an emulator someone started without gRPC. Kept as the multi-touch fallback.
- adb `input` / `motionevent`: single pointer only (AOSP hard-codes `pointerCount = 1`). Used for nothing gestural.
- `sendevent`: same protocol as the console, one process per event, device paths vary. Not used.
- scrcpy-server: real multi-touch, but client/server must match exactly and the protocol is not a public API.
- Appium UiAutomator2: multi-touch, but two server APKs plus an Appium server; slowest start.
- Maestro: no multi-touch (open issue since 2023). mobile-mcp / mobilecli: no pinch, and FSL-1.1 (not open source).
  Midscene pinch: uses YADB (LGPL-3.0).

iOS:

- **WebDriverAgent** (Appium's, BSD-3): public XCTest API; W3C multi-pointer actions give arbitrary two-finger paths;
  `/source` XML our parser already reads; plain HTTP from Node. Appium publishes prebuilt simulator runners
  (`WebDriverAgentRunner-Build-Sim-{arm64,x86_64}.zip`) that need no signing and no xcodebuild on iOS 17+.
- `xcrun simctl`: boot, install, launch, openurl, location, privacy, push, appearance, logs, `io screenshot`,
  `io recordVideo` — but no touch at all.
- idb: MIT and maintained again, pinch added in 2026, but only as a pinch primitive (no two-finger rotate), private
  Indigo HID, and it needs the companion daemon plus a Python or gRPC client.
- AXe: single finger only; private HID; needed a fix for Xcode 27. @expo/serve-sim: arbitrary two-finger, private
  HID, very new. Maestro iOS: no multi-touch.

## Decision

- **Android: adb + the emulator gRPC controller**, console `event send` as the multi-touch fallback. adb does install
  (`install -r`), `am start`, deep links, `uiautomator dump`, `pm grant`, logcat, `gfxinfo` and `meminfo`. Non-ASCII
  text goes through gRPC `setClipboard` + paste, because the proto says key `text` is ASCII only.
- **iOS: `simctl` + prebuilt WebDriverAgent**, launched with `simctl launch` (no xcodebuild). For runtimes older than
  iOS 17, or when a new Xcode breaks the prebuilt runner, the bundled WebDriverAgent source is built once with
  `xcodebuild build-for-testing` and cached per Xcode version.
- The drivers live in `packages/device-driver/src/local/` and are exported Playwright-free as
  `@teststudios/device-driver/local`; the Electron app only hosts them (same seam as the desktop driver).
  Appetize-independent logic (refs, self-heal, waits, scroll-until, accessibility heuristics, variables) lives in
  `NativeDeviceBase`, shared by Appetize and both local sessions.

### Spike results (this Mac: Apple Silicon, emulator 37.1.11, API 36 arm64 AVD, Xcode 26.2, iOS 26.1)

- gRPC `sendTouch` pinch-out on Google Maps zoomed from city to street level; `getevent` showed `ABS_MT_SLOT` 0 and 1
  with two tracking ids — real kernel multi-touch. `getScreenshot` ~85–100 ms.
- Clipboard + `KEYCODE_PASTE` typed "Café 東京 مرحبا" intact. `uiautomator dump` ~2.3 s.
- `-read-only` boot from the quick-boot snapshot in seconds; the discovery file
  (`~/Library/Caches/TemporaryItems/avd/running/pid_*.ini`) advertises `grpc.port` and `grpc.token`.
- `adb emu screenrecord` works on the host (webm) but caps each recording at **180 s** — long runs rotate segments.
- A fresh KaDeep-owned simulator (`simctl create`) boots headless in ~27 s. Prebuilt WebDriverAgent v16.12.11 installs
  and launches through `simctl launch` (port via `SIMCTL_CHILD_USE_PORT`) and is ready in ~6 s, no xcodebuild.
- A W3C two-pointer pinch on Apple Maps zoomed from city to street level. `/source` ~0.5–0.8 s; `simctl io screenshot`
  to stdout and WebDriverAgent `/screenshot` both ~130 ms.

## Hardening that ships with it

- **Clean device per session.** Android boots the chosen AVD with `-read-only`: changes are discarded, the user's AVD
  is never modified, and it can run next to an instance they have open. iOS uses a KaDeep-owned simulator created from
  the chosen device type + runtime; the app is uninstalled between sessions; the user's own simulators are never
  erased.
- **One session per platform per computer.** Suites, matrix runs and scheduled jobs wait for the lease with a visible
  "waiting for <machine>" status and a timeout.
- **Build delivery.** The desktop downloads the build from a short-lived signed API URL (streamed, sha256-checked,
  cached by hash). A build uploaded without an Appetize token is a local-only target (`buildId: local-<sha12>`); an
  Appetize run uploads it on demand.
- **Compatibility preflight.** APK minSdk / native ABIs / Play Services need against the AVD; iOS `.app` archs and
  `MinimumOSVersion` against the host and runtime — a specific message before any boot.
- **Robustness.** `uiautomator dump` fails on screens that never stop animating: retry, then fall back to screenshot +
  coordinates and note it on the run.
- **Safety.** adb only talks to emulator serials the session owns (never a USB phone); the host allow-lists every
  method; there is no generic shell on the host. RAM / disk / acceleration preflight and generous boot timeouts with
  progress.
- **Run video.** iOS `simctl io recordVideo` (mp4, full resolution, ~1.5 MB/s) in 90 s parts; Android host recorder
  (webm, half resolution, 1.5 Mbps) in 180 s parts. Parts attach in order and each stays under the 200 MB evidence
  limit — no ffmpeg is bundled.
- **Rollout.** Behind the org feature flag `localDevices`. Android local is generally available on Mac and Windows;
  iOS local is labelled Beta until the Mac matrix passes on the current and previous Xcode. Runs record
  `device.home` so local vs Appetize use is visible.

## What we do not build

- Take-control on local devices (view-only stream).
- Physical USB devices **through the emulator lane**: the emulator `Adb` still refuses any serial that is not an
  `emulator-N` it started. Real phones are a separate lane with its own rules — see
  [Real-phone lane](#real-phone-lane-agent-device) below.
- Creating AVDs or installing system images / Xcode runtimes: detection shows the exact `sdkmanager`, `avdmanager`
  or `xcodebuild -downloadPlatform iOS` command instead.
- Appium server, Maestro, idb, AXe, scrcpy, mobilecli; YADB-based pinch; `adb input` gestures; `sendevent`.
- Parallel device farms on one machine.

## Gaps (stated on `run.limitations`, never faked)

- No intercepted network traffic locally (Appetize has a proxy, we have none): `device_network` and
  `device_network_assert` return INCONCLUSIVE on the local lane.
- iOS Simulator: no scriptable offline mode, battery or font scale (it shares the Mac's network).
- Long runs arrive as ordered video parts: Android in 180 s parts (the emulator recorder cap), iOS in 90 s parts (simctl records full resolution with no bitrate control, so parts stay under the 200 MB evidence limit).
- Prebuilt WebDriverAgent tracks Appium's release cadence; a brand-new Xcode may need the source build.
- The paired computer's RAM and CPU bound what runs well; the emulator shares the machine with the desktop app.

## Real-phone lane (agent-device)

A third home per platform, `runOn: 'real'`: the user's own Android phone or iPhone plugged into the paired computer by
USB, driven by [agent-device](https://oss.callstack.com/agent-device/docs/introduction) (Callstack, MIT). Same chat,
same `device_*` tools, same run recording, live view, flows, replay and suites; only the engine on the computer
differs.

**Why a lane and not an MCP server.** agent-device's MCP server is stdio-only and the API runs on Railway, so it cannot
reach a USB phone; and tools called through MCP would bypass `tools/mobile.ts` recording (no runs, screenshots,
video, flows). The lane reuses the existing seam instead: `RemoteDeviceDriver` (API) → `device.*` worker calls →
`DeviceSessionHost` (desktop app) → `AgentDeviceSession` (`packages/device-driver/src/real/`).

**How it runs.** Electron bundles Node 20 and agent-device needs Node ≥ 22.12, so it is never imported: every call is
`<node> <agent-device.mjs> <command> … --json --session kadeep-<id> --platform … --serial|--udid …` with no shell, an
allow-list of 24 commands (`mcp`, `proxy` and the rest are refused), a time bound, and the envelope's `error.code` /
`details.reason` turned into a user-facing fix. The CLI is found via `AGENT_DEVICE_BIN`, PATH, nvm / volta / Homebrew /
npm-global, with Node taken from beside it and the Android SDK `platform-tools` added to PATH (an app started from
Finder has neither). Screen reads (`snapshot --json`) go through the shared hierarchy parser, so refs, roles, dialogs,
self-heal and replay behave as on every lane.

| Piece | Where |
|---|---|
| CLI resolution, allow-list, envelope / errors | `packages/device-driver/src/real/agent-device-cli.ts` |
| The driver (`AgentDeviceSession extends LocalDeviceBase`) | `packages/device-driver/src/real/session.ts` |
| Phone detection (`agent-device devices` + `adb devices -l`) | `packages/device-driver/src/real/detect.ts` |
| Capability sheet (`realHardware: true`) | `packages/device-driver/src/real/capabilities.ts` |
| Inventory on the heartbeat, start branch, lease per phone, phone pick | `apps/desktop/src/main/device/{inventory,device-host,policy}.ts` |
| Apple signing team (stays on the computer, `real-devices.json`, mode 600) | `apps/desktop/src/main/device/real-settings.ts` |
| Flag, worker pick by connected phone, re-scan when stale | `apps/api/src/services/local-devices.ts` (`realDevicesEnabled`, `workerCannotHostReal`, `refreshDeviceInventories`) |
| Launch branch, home on runs | `apps/api/src/agent/context.ts`, `apps/api/src/desktop/remote-device.ts` |
| `.ipa` for a real iPhone | `apps/api/src/services/mobile-builds.ts` |
| Run on picker, phone list, setup checklist, composer badge | `apps/web/src/devices/*`, `DeviceLiveView.tsx`, `settings/DevicesSettings.tsx`, `MidPane.tsx` |

**Any project kind.** The lane is the same in QA and Localization projects (screenshots of an app in several languages
for Review → Translations, see [SCREENSHOTS.md](SCREENSHOTS.md#into-localization-review-loccapture-reviewts)). Each project
has its own Run on setting and phone, and a computer is paired to one project at a time: pairing it to another project
moves it there. A project with no computer of its own is told which project of the org has one. A turn is treated as
phone work when the project runs a platform on a real phone and the ask is about the phone, the app or its screens; with
no project builds the agent is still told the phone is there and to use the app already on it.

**Rollout.** Behind its own flag: `REAL_DEVICES=all|off`, else the org setting `features.realDevices`. Off by default,
and `PATCH /devices` refuses `runOn: 'real'` while it is off. It is independent of `localDevices`.

**Requirements.**
- On the computer the phone plugs into: KaDeep Studios (desktop, paired to the project), Node ≥ 22.12 and
  `npm install -g agent-device` (≥ 0.21). Android also needs the Android SDK `platform-tools` (adb).
- Android phone: Developer options → USB debugging on, and "Allow USB debugging" accepted for this computer.
- iPhone: a Mac with Xcode; Developer Mode on (Settings → Privacy & Security), the Mac trusted, the phone unlocked;
  an Apple Team ID entered in the desktop app (agent-device signs its XCTest runner with it; the first run builds it
  and takes a few minutes). Builds must be a signed `.ipa` with this phone in the provisioning profile — simulator
  `.app.zip` builds do not run on hardware.

**Safety rules.**
- The phone is the user's own: nothing is wiped or reset; closing a session ends only KaDeep's agent-device session.
- The phone's system language, text size, network conditions, battery, push, biometrics, shake and interruptions are
  not changed on a real phone — those tools return INCONCLUSIVE (`DeviceCapabilityError`) with alternatives, and the
  agent is told so. An app's own language setting is part of the app: the agent changes it through the app's screens,
  or asks the user to switch the phone's language when the app only follows it.
- Typed text goes on stdin (`fill --text-stdin`), never on a command line; a secret needs a field ref.
- `adb_shell` keeps the existing allow-list and runs only on that phone's serial.
- Leases are per phone, so a phone and the emulator of the same platform can run side by side; two runs never share a
  phone (`BUSY:` and the run waits, as on the emulator lane).
- No silent fallback: no phone, wrong phone, phone not allowed, iPhone on Windows, old desktop app, flag off — each
  stops with the fix.

**Reliability (what keeps it working on any phone).**
- **KaDeep's own agent-device daemon.** The daemon is what runs adb / the iPhone runner, and it keeps the environment
  of whoever started it. KaDeep runs agent-device with `AGENT_DEVICE_STATE_DIR=<desktop userData>/agent-device`, so its
  daemon always has KaDeep's PATH, adb wrapper and signing team, and an older daemon left by Cursor's MCP or a terminal
  (`~/.agent-device`, "Daemon (vX) is newer than this client", "Not connected") cannot break it. The daemon is
  restarted — only when no KaDeep phone session uses it — when agent-device is upgraded or those settings change
  (`real/daemon.ts`), and stopped when the desktop app quits. Device claims stay host-wide (agent-device keeps them in
  `~/.agent-device/device-claims` for every daemon), so a phone held by another tool is still seen and never driven twice.
- **adb wrapper for multi-user phones.** On phones with a second Android user (Samsung Secure Folder, work profiles,
  Xiaomi Second space, dual apps) `pm list packages` / `cmd package list packages` fail without `--user 0`. agent-device
  0.21.x reads that as "snapshot helper missing" and reinstalls it on every screen read (30 s timeout each). KaDeep's
  daemon uses an `adb` wrapper (`real/adb-shim.ts`) that adds `--user 0` to package listings only; everything else goes
  straight to the real adb.
- **Phone preparation.** agent-device's helpers (Android screen reader + IME helper, iPhone XCTest runner) are read from
  agent-device's own manifests and installed up front — at session start (required helper only) or with **Prepare
  phone** (all) — with a 3-minute budget and on-screen guidance for install / Play Protect prompts (`real/prepare.ts`).
- **One line of work per phone.** Agent / user steps run in order; live-view frames run only when the phone is idle
  (Android frames come straight from `adb screencap`) and slow down when the phone is slow (`real/queue.ts`).
- **One session per phone, and a chat always gets it.** Our own session is reused. A chat asking for a phone (or
  emulator) another chat or run holds is handed it: that session is ended and its chat told ("handed to another chat");
  scheduled / MCP runs still wait. Anything else holding the phone is freed when a session starts (`real/ownership.ts`
  `freePhone`): a stale claim through agent-device's own `device release --stale`; a KaDeep leftover in KaDeep's
  daemon (restarted when it forgot the session); another tool's session (Cursor, a terminal) closed in its own daemon.
- **Leftover claims.** agent-device keeps a phone's host-wide claim when a session's resource cannot be settled — seen
  with an Android recording whose recorder had stopped and whose process id Android gave to another app ("screenrecord
  ownership could not be confirmed"). The claim then outlives KaDeep's daemon: `devices` hides it (stale), `open` still
  refuses. Every phone scan frees such KaDeep leftovers on plugged-in phones; for an unfinished recording it first proves
  no `screenrecord` writes the session's files, removes the phone-side marker and parts, and sets the session's
  `screen-recording.resource.json` aside (renamed, kept). A failed close is followed by the same cleanup at once.
  Setup lists stale and foreign claims (`device status [--stale]`) with **Release phone**, which runs the same steps.
- **Deadlines and health check.** One table of deadlines (`DEADLINES`); a step that runs out of time is followed by one
  health check (phone gone → "disconnected"; helper gone → installed again), and the agent is told to stop after the
  same device error twice. Errors never show command lines or paths.
- **Same build not reinstalled.** The desktop records which build (sha256) it installed on which phone and checks the
  phone still has the app before skipping the install.
- **Check any phone:** `npm run test:real-phone -- --platform android|ios [--serial|--udid …] [--apk|--ipa …]` drives the
  phone through this lane (prepare, open Settings, read the screen twice, screenshot, scroll, tap and back, type when a
  field exists, rotate, home, relaunch, logs, optional install, close with video) and prints a pass / fail table. It only
  navigates. Tested agent-device range: 0.21.x – 0.22.x (the setup panel flags others).
- Transport: KaDeep calls the agent-device CLI (one process per step; agent-device cancels the daemon request when the
  process is stopped). A persistent bridge on agent-device's typed client was considered; it is not used because the
  client exposes no raw request API and mapping every command onto typed options could not be verified on hardware —
  the per-phone queue removed the per-frame process cost that mattered.

**Troubleshooting.**

| Symptom | Cause | Fix |
|---|---|---|
| "Device is already in use by session "default"" / phone shows **In use** | Cursor's agent-device MCP or a terminal command (no `--session`) holds the phone | **Release phone** in Mobile → Run on → Setup, or `agent-device close --session default --platform android` |
| "The phone is held by a KaDeep session that ended without releasing it" | A KaDeep session's close failed (an unfinished recording) and its daemon has since stopped | Freed by the next phone scan or session start; **Release phone**, or `agent-device device release --platform android --serial <serial> --stale`. `agent-device device status --stale` shows such claims |
| `Android snapshot helper failed: adb timed out after 30000ms` (outside KaDeep) | Multi-user phone (Secure Folder): `pm list packages` needs `--user 0` | Inside KaDeep the adb wrapper handles it; for Cursor's MCP, remove the Secure Folder user or wait for an agent-device fix |
| "Daemon (vX) is newer than this client" | A daemon from another agent-device version still runs | KaDeep restarts its own when idle; for the shared one: `agent-device daemon stop --state-dir ~/.agent-device`, then reload the Cursor MCP |
| "Preparing … installing the helper" waits | The phone shows an install / Play Protect / "Install via USB" prompt | Accept it on the phone, keep it unlocked |
| Phone missing from the list | Cable / USB mode / debugging permission | Replug, choose File transfer, accept "Allow USB debugging", press Check again |

**Gaps.** No intercepted network traffic (as on the emulator lane); frame and memory performance metrics are not
collected yet (settle time only); the live view is view-only and polls screenshots (~1–2 fps); devices not listed by
agent-device but seen by adb are shown with "run agent-device doctor".

## What stays on Appetize

- iOS for Windows users and for Macs without Xcode; everything for Windows-on-ARM users.
- Device models or OS versions not installed locally.
- Parallel scale beyond one machine.
- Network interception and proxy evidence.
- The manual Sandbox embed and take-control.
- Any platform whose setting is left on Appetize (the default).
