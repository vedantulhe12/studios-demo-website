# Building KaDeep Studios installers

KaDeep Studios ships as four installers, all produced by one GitHub Actions workflow —
**Build artifacts** (`.github/workflows/build-artifacts.yml`). Nothing runs on a
schedule or on push: you start a build when you want one, and every installer from
that run lands on a single GitHub **Release** with a checksum manifest.

| Platform | What you get | Runner |
|----------|--------------|--------|
| Android  | `KaDeepStudios-<version>-android.apk` | Ubuntu |
| iOS      | `KaDeepStudios-<version>-ios.ipa` (signed) or `…-ios-unsigned.ipa` | macOS |
| macOS    | `KaDeepStudios-<version>-macos-arm64.dmg` | macOS |
| Windows  | `KaDeepStudios-<version>-windows-x64-setup.exe` and `…-portable.exe` | Windows |

`<version>` comes from `apps/desktop/package.json`.

## 1. Start a build

### From the GitHub UI

1. Open the repository on GitHub → **Actions** tab.
2. In the left sidebar pick **Build artifacts**.
3. Click **Run workflow** (top right), fill in the options and click the green **Run workflow** button.

### From the terminal

Requires the [GitHub CLI](https://cli.github.com) (`gh auth login` once).

```bash
# everything, with defaults
gh workflow run build-artifacts.yml

# only what you need
gh workflow run build-artifacts.yml -f targets=android
gh workflow run build-artifacts.yml -f targets=desktop      # macOS + Windows
gh workflow run build-artifacts.yml -f targets=mobile       # Android + iOS

# point the mobile app at a different API and publish a GitHub Release
gh workflow run build-artifacts.yml \
  -f api_url=https://api.staging.teststudios.ai \
  -f release_tag=v1.1.2

# follow progress
gh run watch
```

### Options

| Input | Default | Meaning |
|-------|---------|---------|
| `targets` | `all` | `all`, `android`, `ios`, `macos`, `windows`, `mobile` (Android + iOS) or `desktop` (macOS + Windows). |
| `api_url` | `https://api-production-073b.up.railway.app` | API base URL compiled into **desktop** (`TESTSTUDIOS_API_URL`) and **mobile** (`EXPO_PUBLIC_API_URL`). Used as-is when `target` is `default`. |
| `target` | `default` | `default` keeps `api_url`. `dev` bakes `https://grip.kadeep.ai`. `production` bakes `https://api.kadeep.ai`. A different `api_url` still wins. A dev desktop build opens a sign-in window when that host asks for Microsoft login, then keeps the cookie for the app. Production builds are not given that host. |
| `update_url` | *(empty)* | Desktop update feed origin baked as `KADEEP_UPDATE_URL`. Empty leaves **Check for Updates** off, unless repository variable `DESKTOP_UPDATES_URL_DEV` or `DESKTOP_UPDATES_URL_PRODUCTION` is set for that `target`. |
| `release_tag` | *(empty)* | Tag of the Release to publish to (e.g. `v1.1.2`), created if missing or updated if it exists. Leave empty to publish a `build-<run number>` **pre-release** — the usual choice for test builds. The update feed moves only when this tag is `v` plus the version in `apps/desktop/package.json`. |

Builds for the selected platforms run in parallel. Expect roughly 10–15 minutes for
desktop-only runs and 25–40 minutes when iOS/Android are included (the first run is
slower while Gradle/CocoaPods caches warm up).

## 2. Download the result

Every run publishes to **Releases** (`https://github.com/<org>/teststudios.ai/releases`):

- `build-<run number>` pre-release for a normal run, or the `release_tag` you gave.
- Assets: every installer that was built plus `MANIFEST.md` (file list, sizes, SHA‑256
  checksums, per‑platform status, commit and inputs). The manifest is also the release
  description and is shown in the run's summary page.

From the terminal:

```bash
gh release list                                   # find the tag
gh release download build-42 -D ./installers      # everything from that run
gh release download build-42 -p '*.dmg'           # just one platform
```

The run page's **Artifacts** section also gets per‑platform copies and a
`KaDeepStudios-<version>-all-platforms-<run>` bundle — *when quota allows*. Those are
best‑effort with short retention: a full run is over 1 GB and Actions artifact storage
on private repos is limited, so the Release is the source of truth.

### Releases page

`releases-dev.kadeep.ai` lists the last three development builds. `releases.kadeep.ai` lists the last three production builds. Both are the desktop update service, with a password in `RELEASES_PASSWORD` on that Railway service. Set `RELEASES_CHANNEL` to `dev` on the development service. On production, leave it unset or set `production`. The page reads the Mac zip (or the disk image, when that file is in the bucket) and the Windows setup already stored there. It does not rebuild the desktop app. `/stable/` stays open so Check for Updates keeps working.

If one platform fails, the others still publish and the manifest marks the failed one —
the run is marked failed so you notice, but nothing built is lost.

## 3. Installing

**Android (APK)** — copy to the device and open it, or `adb install KaDeepStudios-*.apk`.
Allow "install from unknown sources" when prompted.

**iOS (IPA)** — a *signed* IPA (see secrets below) installs on the devices listed in the
provisioning profile via Finder, Apple Configurator or an MDM. An *unsigned* IPA must be
re‑signed first (e.g. with Sideloadly, AltStore or `codesign` + your own profile).

**macOS (DMG)** — open the DMG and drag the app to Applications. Builds are not notarised.
The Terminal icon at the top of the window opens a text file with one command to paste
into Terminal; that clears the download quarantine and opens the app. You can also
right‑click the app and choose **Open**. macOS will not run an unsigned command straight
from the disk image. The installer is Apple Silicon (`arm64`) only. Dev builds use
**KaDeep Studio Pro Dev** in that command; production builds use **KaDeep Studios**.

**Windows (EXE)** — run the `-setup.exe` installer (choose install location, creates
Start‑menu and desktop shortcuts) or just launch the `-portable.exe`. The binary is not
code‑signed, so SmartScreen will show **More info → Run anyway** the first time.

## 4. Signing (optional)

Every build works with **no secrets configured**. Adding these repository secrets
(**Settings → Secrets and variables → Actions**) upgrades the output:

| Secret | Effect |
|--------|--------|
| `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD` | APK is signed with your release keystore instead of the Expo debug keystore (required for Play Store / upgrading over an existing install). `base64 -i my.keystore \| pbcopy` to produce the first value. |
| `IOS_P12_BASE64`, `IOS_P12_PASSWORD`, `IOS_PROVISIONING_PROFILE_BASE64`, `IOS_TEAM_ID` | Produces a signed **ad‑hoc** IPA for bundle id `ai.teststudios.mobile` that installs directly on the devices in the profile. Export the distribution certificate + key as a `.p12` from Keychain Access and download the ad‑hoc profile from developer.apple.com. |

macOS and Windows installers stay unsigned unless you add the signing secrets below **and** set `release_tag` to `v` plus the desktop version. A `target=dev` build publishes its zip to the dev update feed only, and a `target=production` build publishes to the production feed only. A default build joins neither feed. Dev builds are a separate app, **KaDeep Studio Pro Dev**, so they can sit next to production.

| Secret | Effect |
|--------|--------|
| `CSC_LINK`, `CSC_KEY_PASSWORD` | Developer ID Application `.p12` (base64 or a URL electron-builder can fetch) and its password. Signs the Mac app. Also signs Windows when `WIN_CSC_LINK` is unset. |
| `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID` | Notarize the signed Mac app. All three are required before a Mac build is treated as signed. |
| `WIN_CSC_LINK`, `WIN_CSC_KEY_PASSWORD` | Authenticode certificate for Windows, when it is not the same file as `CSC_LINK`. |
| `DESKTOP_UPDATES_BUCKET`, `DESKTOP_UPDATES_ACCESS_KEY_ID`, `DESKTOP_UPDATES_SECRET_ACCESS_KEY`, `DESKTOP_UPDATES_ENDPOINT`, `DESKTOP_UPDATES_REGION` | Write credential for the **desktop updates** bucket only. Not the evidence bucket. The feed service on Railway uses a read of the same bucket. If any of these or the signing secrets are missing, the feed step logs a skip and the GitHub Release still succeeds. |

## 4b. Desktop worker mode and the cua-driver binary

The desktop app doubles as the **desktop worker** (`docs/DESKTOP.md`). The **Build artifacts** workflow fetches
`cua-driver` for the target OS (`fetch-cua-driver.mjs darwin-arm64` / `windows-x86_64`) before `electron-builder`.
Locally: `node apps/desktop/scripts/fetch-cua-driver.mjs all` (or one `<os-arch>`); binaries land in
`apps/desktop/resources/cua-driver/` (git-ignored) and electron-builder copies them under `extraResources`. Without
them the app still runs and falls back to a driver installed on the machine (`~/.local/bin/cua-driver`, `PATH`).
Windows CI asserts `cua-driver.exe` exists after fetch and again inside `win-unpacked/resources`. If that binary is
still absent at runtime the app downloads the pinned GitHub zip itself — do not send people to the cua.ai PowerShell
installer (Windows blocks scripts by default).

- `KaDeepStudios --worker` starts headless (tray + overlay only) for a secondary test machine.
- Pair from the **Desktop** tab (**Pair this computer** inside the app) or `kadeepstudiospro://worker/pair?token=…&api=…`.
- Updates: the Mac app menu and the Windows **Help** menu (and the tray) offer **Check for Updates…**. A packaged app checks the feed baked from `update_url` on launch and every 6 hours, downloads in the background, and installs when you choose **Restart** or when you quit. It will not restart while a desktop, device, or agent session is running. The first build that contains this menu is still installed from the DMG or setup.exe. Later signed builds, published with `release_tag` equal to `v<version>`, replace themselves. Unsigned builds are not put on the feed. On macOS the app must live in `/Applications`; the first launch of a copy that is still on the disk image offers to move it. Bump `apps/desktop/package.json` before that release. The feed service is `services/desktop-updates` (Dockerfile). It reads `stable/macos.json` and `stable/windows.json` from the updates bucket and redirects each download to a short-lived bucket URL. Deploy that service only in the Railway dev environment until you are ready to point production at its own bucket.

## 5. Building locally

```bash
npm install

# Desktop (from the repo root) — output in apps/desktop/release/
npm run dist:mac                    # unpacked .app for a quick check (macOS only)
npm run dist:mac -- dmg             # .dmg, like CI
npm run dist:win -- nsis portable   # setup + portable .exe (Windows, or macOS with Wine)

# Mobile — see apps/mobile/README.md for EAS / local Expo builds
npm run build:apk -w @teststudios/mobile
npm run build:ipa -w @teststudios/mobile
```

### 5a. Android APK without EAS, Android Studio or secrets (894)

```bash
./scripts/setup-android-toolchain.sh      # JDK 17 (if no 17/21), cmdline-tools, platform 36, build-tools, NDK 27 → ~/android-sdk
EXPO_PUBLIC_API_URL=https://api-production-073b.up.railway.app ./scripts/build-apk-local.sh
# → dist/KaDeepStudios-<version>-android.apk (+ .sha256), signed with the Expo debug keystore
```

The setup script also installs `~/.gradle/init.d/central-mirror.gradle`: Maven Central rate-limits shared cloud
egress addresses (HTTP 429, which Gradle treats as a hard failure), so Central is resolved through Google's mirror
first. Delete the file if you prefer the default resolution. The APK installs on any Android device or emulator
and uploads to Appetize (Mobile → Builds, or `install_app` in chat) so the agent can dogfood the companion app —
with `APPETIZE_API_TOKEN` set, drop the APK into a chat and say *"install this build, boot a Pixel 8 Pro, sign in
with the vault user and smoke-test every tab; screenshots as evidence"* (the `mobile-smoke` skill drives it). The
platform's own intake (`inspectMobileBuild`) recognises the file as `ai.teststudios.mobile`, runnable;
set `ANDROID_KEYSTORE` / `ANDROID_KEYSTORE_PASSWORD` / `ANDROID_KEY_ALIAS` / `ANDROID_KEY_PASSWORD` to sign with your own key.

### 5b. Desktop from a Linux box

electron-builder packages the Linux worker natively and can package macOS targets *except* DMG (which needs
`hdiutil`); signing needs a Mac. What a Linux machine can produce:

```bash
cd apps/desktop
TESTSTUDIOS_API_URL=https://api-production-073b.up.railway.app npx electron-vite build
node scripts/fetch-cua-driver.mjs linux-x86_64 darwin-arm64
npx electron-builder --linux AppImage --x64 --publish never                 # KaDeepStudios-<v>-linux-x86_64.AppImage (worker for Linux desktops)
npx electron-builder --mac zip --arm64 --publish never -c.mac.identity=null # KaDeepStudios-<v>-macos-arm64.zip (unsigned .app; DMG comes from the workflow)
```

### 5c. iOS

The IPA needs Xcode, so it stays a workflow job. The workflow also produces
`KaDeepStudios-<version>-ios-simulator.app.zip` (job **iOS simulator app**): an unsigned simulator build that
Appetize accepts, so the companion app can be dogfooded on an iPhone simulator without a developer account.

### 5d. Push notifications on the companion app

Expo push tokens need an EAS `projectId` in `apps/mobile/app.json` (`extra.eas.projectId`). Without it the app
runs fine but `registerForPush` reports "push not configured" and the Inbox relies on polling; the Settings screen
shows that state. Create the project once with `eas init` (needs an Expo account) and commit the id.

## Troubleshooting

- **"Run workflow" button is missing** — the workflow file must exist on the default
  branch; push it to `main` first. If the Actions tab is empty altogether, Actions are
  disabled for the repo/org (**Settings → Actions → General**).
- **"Artifact storage quota has been hit"** on an upload step — the installers are still
  on the Release; only the optional workflow artifact was skipped. Delete old artifacts
  under **Actions → Management → Caches/Artifacts** if you want them back; the quota is
  recalculated every 6–12 hours.
- **Too many `build-N` pre-releases** — delete them freely (`gh release delete build-N --cleanup-tag -y`);
  each is independent.
- **Android job fails in Gradle** — usually a dependency resolution hiccup; re‑run the
  failed job. Genuine build errors reproduce locally with
  `cd apps/mobile && npx expo prebuild -p android && cd android && ./gradlew assembleRelease`.
- **iOS job fails in `pod install`** — re‑run; if it persists, bump the Xcode/macOS runner
  image in the workflow.
- **Signed iOS build fails with "no profile"** — the profile's bundle id must be
  `ai.teststudios.mobile` and the certificate in the `.p12` must be the one the profile
  was created for.
