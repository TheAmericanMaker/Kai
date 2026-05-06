# Architecture Map — Kai 9000

## System Intent

Kai 9000 is an open-source AI assistant with persistent memory that runs as a single Kotlin
Multiplatform / Compose Multiplatform application across **Android, iOS, Windows, macOS, Linux,
and Web (WASM)**. *(observed fact — README.md §intro, settings.gradle.kts module list, source-set
directories under `composeApp/src/`).* The product binds 24+ LLM provider integrations behind a
fallback chain, ships an autonomous heartbeat (periodic self-check that surfaces follow-ups via
Android notifications), supports Model Context Protocol servers as a tool transport, persists
conversations in encrypted local storage, and — uniquely on Android — embeds a **bundled
Alpine-Linux + proot sandbox** that the assistant can shell into to run real commands. *(observed
fact — README.md §Features, §Linux Sandbox)*

System purpose, in one line: "ship one shared agent core to every consumer device, with an
on-device tool-execution sandbox where the host OS allows it." *(strong inference — the
multi-target source-set layout plus the `expect/actual` `SandboxController` interface plus the
Android-only `LinuxSandboxManager` implementation.)*

## Layer Map

### Package Inventory

The repo is a Gradle multi-module project. `settings.gradle.kts` defines three Gradle modules
(`:composeApp`, `:androidApp`, `:screenshotTests`); the iOS surface lives in a sibling Xcode
project (`iosApp/`) consumed via Kotlin/Native binary frameworks built by `:composeApp`.

| Package / Module | Role | Public Entrypoints | Key Dependencies | Runtime Surface |
|---|---|---|---|---|
| `:composeApp` (commonMain) | core semantics + cross-platform UI | `App()` Composable, `SandboxController` interface, ViewModels (`ChatViewModel`, `SandboxPackagesViewModel`, …), `DataRepository`, settings/heartbeat/tool layers | Compose Multiplatform, Koin DI, Ktor Client, kotlinx.coroutines/serialization, `io.github.vinceglb.filekit` (file pickers) | shared by every platform |
| `:composeApp` (androidMain) | platform adapter — Android | `AndroidSandboxController`, `LinuxSandboxManager`, `RootfsDownloader`, `ProotExecutor`, `DaemonService`, `AndroidDaemonController`, `HeartbeatNotifier`, `KaiNotificationListenerService`, `ModelDownloadService`, `NotificationHelper`, `LiteRTInferenceEngine` (via `jvmShared`) | Android SDK, Ktor `engine.android.Android`, FileKit Android, Koin Android, MediaPipe / LiteRT | Android process |
| `:composeApp` (iosMain) | platform adapter — iOS | `actual` overrides for sandbox (no-op), platform, file kit | Kotlin/Native, FileKit iOS | iOS framework consumed by `iosApp/` |
| `:composeApp` (desktopMain) | platform adapter — JVM desktop | `NoOpSandboxController`, JVM platform helpers, FileKit Compose desktop | Compose Desktop, Java Standard Library | macOS / Windows / Linux JVM app |
| `:composeApp` (wasmJsMain) | platform adapter — Web (WASM) | `NoOpSandboxController`, JS platform helpers, FileKit Web | Compose Web, kotlin-wrappers | browser |
| `:composeApp` (jvmShared) | JVM-only shared code (intermediate source set used by both Android and Desktop targets) | `LiteRTInferenceEngine` | LiteRT (TensorFlow Lite) | Android + JVM Desktop |
| `:androidApp` | product shell — Android | `MainActivity`, `KaiApplication`, `AndroidManifest.xml`, manifest-declared services (`DaemonService`, `ModelDownloadService`), build flavors `foss` and `playStore` | depends on `:composeApp`; pulls in proot/talloc native libs from `androidApp/src/main/jniLibs/{arm64-v8a,armeabi-v7a,x86_64}/` | Android APK |
| `iosApp/` | product shell — iOS | Swift `iosApp` Xcode project that hosts the Kotlin Multiplatform framework | depends on `:composeApp` Kotlin/Native framework | iOS App |
| `:screenshotTests` | testing — UI screenshot harness | Roborazzi-style screenshot tests | Compose test, `:composeApp` | dev/CI only |
| `tools/finetuning/` | tooling — fine-tune golden capture & output prep | Python scripts (out-of-band) | n/a | dev only |
| `flatpak/`, `aur/`, `fastlane/`, `site/` | distribution packaging / website | Flatpak manifest, AUR PKGBUILD, Fastlane metadata, mkdocs site | external | release pipeline |

*(observed fact — `settings.gradle.kts`, `composeApp/src/<sourceSet>/kotlin/...` tree, manifest
declarations, `build-proot.sh` output paths, `tools/finetuning/`, `mkdocs.yml`, `flatpak/`,
`aur/`, `fastlane/` directories.)*

### Dependency Direction

Stable base → leaf:

1. **commonMain** (no dependency on platforms; defines `expect` declarations including
   `createSandboxController()`, `Platform`, file/heartbeat/notifier abstractions).
2. **jvmShared** depends on commonMain; consumed by androidMain and desktopMain (LiteRT inference
   engine).
3. **androidMain / iosMain / desktopMain / wasmJsMain** each depend on commonMain (plus jvmShared
   where applicable). They provide `actual` implementations and platform-specific subsystems.
4. **`:androidApp`** depends on `:composeApp`. It owns the Android process entry point
   (`MainActivity`, `KaiApplication`) and the Android-only native libraries.
5. **`iosApp/`** depends on `:composeApp` (Kotlin/Native framework).
6. **`:screenshotTests`** depends on `:composeApp` (test surface only).

No cycles observed. The shared agent and UI behavior live in commonMain; platform adapters fan
out beneath; product shells (`:androidApp`, `iosApp`) sit at the leaves. `composeApp/androidMain`
is the *thickest* adapter layer because of the Linux sandbox and the foreground-service daemon —
neither has a counterpart on iOS, desktop, or web. *(strong inference — settings.gradle.kts plus
direct reads of platform `actual` declarations: `SandboxController.android.kt`, `Platform.kt`
expect/actuals.)*

## Public Surfaces

- **Android user-facing surfaces:** `MainActivity` Compose UI (chat, settings, sandbox terminal,
  heartbeat history, package manager UI, file browser); foreground notification ("Kai daemon
  running"); heartbeat notifications; AI-driven `send_notification` tool notifications;
  notification-listener service for reading other apps' notifications. *(observed fact —
  `AndroidManifest.xml`, `MainActivity.kt`, `DaemonService.kt`, `HeartbeatNotifier.android.kt`,
  `NotificationHelper.kt`, `KaiNotificationListenerService.kt`.)*
- **iOS / Desktop / Web user-facing surfaces:** Compose UI for chat / settings / heartbeat
  history. Sandbox features are *advertised but no-op* on these platforms (the
  `NoOpSandboxController` returns failures with the message "Sandbox file browser is
  Android-only"). *(observed fact — `SandboxController.jvm.kt:26`,
  `SandboxController.wasmJs.kt:26`.)*
- **External integrations Kai consumes (not surfaces it exposes, listed for porting clarity):** 24+
  LLM HTTP/SSE provider APIs (Anthropic, OpenAI, Gemini, DeepSeek, Mistral, Ollama, …); arbitrary
  user-supplied MCP servers; Android-specific: TextToSpeech (Google engine), Calendar provider,
  notification listener, FileKit-backed file pickers. *(observed fact — README.md §Features;
  `MainActivity.kt:73` initializes `rememberTextToSpeechOrNull(TextToSpeechEngine.Google)`;
  manifest READ_CALENDAR/WRITE_CALENDAR permissions.)*
- **Wire formats Kai produces / consumes:** Settings export/import (JSON file); encrypted local
  conversation storage; per-session shell transcripts (in-memory, optionally persisted for
  chat-bound sessions per `SandboxSessions.isPersistable`); MCP tool-call JSON-RPC. *(observed
  fact — `SandboxController.kt:51`, README.md §Features list, `tools/finetuning/golden/`.)*

## Runtime Lifecycle

- **Android entry:** `KaiApplication.onCreate()` (Application class — referenced from manifest;
  bootstraps Koin DI). `MainActivity.onCreate()` then runs `FileKit.init(this)`,
  `enableEdgeToEdge()`, `handleDeepLinkIntent()`, and renders `App()`. `MainActivity.onStart()`
  re-asserts the daemon by calling `AndroidDaemonController.start()` if `appSettings
  .isDaemonEnabled()` — explicitly **idempotent and re-asserted on every foreground** to recover
  from aggressive OEM battery managers (MIUI, EMUI, Huawei) silently killing the foreground
  service. *(observed fact — `MainActivity.kt:33-100`, code comment lines 91-100.)*
- **Daemon foreground service:** `DaemonService.onCreate()` creates a low-importance notification
  channel, calls `startForeground(NOTIFICATION_ID=9001, …)`, then starts `taskScheduler.start()`.
  `onStartCommand` returns `START_STICKY` so the OS re-creates the service after kills.
  `onTimeout()` (Android 14+ FGS time-limit handler) cleanly stops the service.
  `onDestroy()` calls `stopForeground(STOP_FOREGROUND_REMOVE)`. *(observed fact —
  `DaemonService.kt:23-52`.)*
- **Sandbox lifecycle (Android only):** `LinuxSandboxManager` exposes a sealed
  `StateFlow<SandboxState>`: `NotInstalled → Downloading(progress) → Extracting → Installing(detail)
  → Ready / Error(message)`. State transitions are driven by `setup()` → tar download → tar
  extract → post-install (resolv.conf write, repositories file write, makeWritable walk) →
  `Ready`. `cancel()` can interrupt `Downloading` / `Extracting`. `reset()` wipes the rootfs.
  `installPackages()` triggers package-manager install via the proot shell. *(observed fact —
  `SandboxController.android.kt:78-121` mapping; `LinuxSandboxManager.kt` state field;
  `RootfsDownloader.kt:217-234` post-install helpers.)*
- **iOS / Desktop / Web entry:** Standard Compose entry points (`SwiftUI` host on iOS; `Window`
  composable on desktop; `CanvasBasedWindow` on wasmJs). No daemon, no sandbox, no foreground
  service — these targets surface a strict subset of features. *(strong inference — source-set
  layout plus `NoOpSandboxController`.)*

## Concurrency Model

- Primary primitive: **Kotlin coroutines + StateFlow** throughout commonMain (ViewModels) and
  androidMain (sandbox + daemon). `CoroutineScope(SupervisorJob() + Dispatchers.IO)` is the
  recurring shape for IO-bound work, with `withContext(Dispatchers.IO)` wrappers on the
  `SandboxController` blocking calls (file IO, shell exec). *(observed fact —
  `SandboxController.android.kt:29, 149, 208, 268`; `LinuxSandboxManager.kt:25`.)*
- **proot subprocess management:** `ProotExecutor.execute()` calls `Runtime.exec(...)` and uses
  two `CompletableFuture.supplyAsync` readers on `process.inputStream` and
  `process.errorStream` to drain in parallel — explicit comment cites pipe-buffer-deadlock
  avoidance. Cancellation polls `process.isAlive` and `cancelled.get()` every 200 ms because on
  Linux `close(fd)` does not unblock a thread inside `read(fd)`. *(observed fact —
  `ProotExecutor.kt:30-47, 66-82`.)*
- **Persistent shell sessions:** `SessionShell` (per-`sessionId`) keeps a long-lived proot
  process; `executeCommandStreaming` runs commands inside it with no implicit timeout in the
  streaming path (cancellation is the real "done" signal). The 30 s default timeout in
  `executeCommand` is bypassed in streaming. *(observed fact — `SandboxController.android.kt:177-205`.)*
- **Portability hazards** (concurrency-flavored): the proot drain pattern is intrinsically tied
  to Java `Process` + Linux pipe semantics. The `CompletableDeferred` + `AtomicBoolean` cancel
  flag pattern is JVM-specific. Re-implementations on iOS / desktop / web don't have to deal
  with this because the sandbox is no-op there. *(strong inference — `ProotExecutor.kt:80-82`
  comment; absence of sandbox on non-Android.)*

## Build and Packaging

- **Build system:** Gradle Kotlin DSL with `libs.versions.toml` version catalog (Kotlin
  Multiplatform, Compose Multiplatform, Compose Compiler, Android Application/Library, Spotless).
  Spotless+KtLint enforces formatting on `**/*.kt` and `**/*.gradle.kts`. *(observed fact —
  `build.gradle.kts`, `settings.gradle.kts`, `gradle/libs.versions.toml`.)*
- **Native build for Android:** `build-proot.sh` (350 lines, bash) cross-compiles
  [termux/proot](https://github.com/termux/proot) at pinned commit `4dba3afbf3a63af89b4d9c1a59bf2bda10f4d10f`
  + talloc 2.4.3 against Android NDK r29 for `arm64-v8a / armeabi-v7a / x86_64`. Outputs
  `libproot.so`, `libproot-loader.so`, `libproot-loader32.so`, `libtalloc.so` into
  `androidApp/src/main/jniLibs/<abi>/`. The 32-bit loader is built separately because NDK clang
  can't `-m32`. `.comment` sections are stripped post-link for F-Droid reproducible-build
  parity. *(observed fact — `build-proot.sh:18-27, 53-94, 269-321, 333-340`.)*
- **GitHub Actions:** `test.yml`, `release.yml` (13 KB, full release pipeline), `static.yml`
  (linting), `flatpak.yml`, `aur.yml`, `winget.yml`. *(observed fact — `.github/workflows/`
  listing.)*
- **Distribution channels:** Google Play, F-Droid, Apple App Store, Homebrew (`brew install
  --cask simonschubert/tap/kai`), AUR (`yay -S kai-bin`), winget (`winget install
  SimonSchubert.Kai`), Flatpak, AppImage, DEB, RPM, MSI, DMG, web build. *(observed fact —
  README.md §Installation, `aur/`, `flatpak/`, `fastlane/`, GitHub Actions release workflow.)*
- **Detailed build/deploy notes:** appended to `findings/build-and-deploy/build-and-deploy.md`
  for porting reference.

## Porting Priorities

| Component | Priority | Rationale |
|---|---|---|
| Multi-LLM provider chain + fallback | core | Required for Kai to do anything useful — every product surface depends on it. |
| Persistent memory layer | core | Identity feature; conversations + memory are the differentiator. |
| Compose Multiplatform UI shell | core | Carries the same UX across all platforms. |
| Android foreground daemon (`DaemonService` + `TaskScheduler`) | important | Required for autonomous heartbeat on Android; no parallel exists for iOS, desktop, web (those run heartbeat only while the app is open). *(portability hazard — Android FGS rules don't translate to other OSes.)* |
| Linux sandbox (Alpine + proot) | important on Android, deliberate non-goal elsewhere | Big behavioral differentiator on Android; explicitly stubbed (`NoOpSandboxController`) on every other target. *(portability hazard — proot is Linux-only and depends on `ptrace`; iOS/desktop/web require a different sandbox primitive or no sandbox.)* |
| MCP server tool transport | important | Wraps remote tools, used by chat. |
| LiteRT on-device inference (Android, JVM) | optional | Falls back to cloud LLMs if missing. |
| TextToSpeech (Google engine) | optional | UX polish. |
| Notification listener (Android) | optional | Read-other-apps' notifications feature. |
| Build-from-source proot pipeline (`build-proot.sh`) | core *for Android packaging* | Required for F-Droid reproducible builds; CI must run it. |
| Per-platform packaging (Flatpak/AUR/winget/Homebrew/etc.) | optional | Convenience for downstream consumers. |

---

## Focus subsystem deep-dives (per status.yaml carry_forward)

### Alpine / proot bootstrap

**Where it lives:**
- Native binaries: built by `build-proot.sh` from termux/proot @ pinned commit
  `4dba3afbf` + talloc 2.4.3, output to `androidApp/src/main/jniLibs/{arm64-v8a,armeabi-v7a,x86_64}/`
  as `libproot.so`, `libproot-loader.so`, `libproot-loader32.so`, `libtalloc.so`. *(observed
  fact — `build-proot.sh:18-27, 256-265`.)*
- Rootfs download + extract: `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt`.
  Downloads `alpine-minirootfs-3.21.3-<arch>.tar.gz` from one of six mirrors (dl-cdn.alpinelinux.org,
  mirrors.edge.kernel.org, ftp.halifax.rwth-aachen.de, alpine.ethz.ch, mirror.csclub.uwaterloo.ca,
  mirrors.tuna.tsinghua.edu.cn) with sequential fallback on per-mirror failure. *(observed fact
  — `RootfsDownloader.kt:17-27, 47-64`.)*
- State machine: `composeApp/src/androidMain/.../sandbox/LinuxSandboxManager.kt` (363 lines).
  Holds a `MutableStateFlow<SandboxState>` with sealed `NotInstalled / Downloading /
  Extracting / Installing / Ready / Error` states. Sandbox storage at
  `context.filesDir/linux-sandbox/`; rootfs at `<sandbox>/rootfs`; `/root` is bind-mounted from
  `getExternalFilesDir(null)/sandbox-home` so files produced inside the sandbox are visible to
  Android FileProvider for opening with system viewers. *(observed fact —
  `LinuxSandboxManager.kt:30-67`.)*
- proot invocation: `composeApp/src/androidMain/.../sandbox/ProotExecutor.kt`. Runs
  `applicationInfo.nativeLibraryDir/libproot.so` via `Runtime.exec(...)`; default 30 s timeout,
  hard cap 180 s for one-shot exec; output bounded to 15 KB to prevent memory blowup. *(observed
  fact — `ProotExecutor.kt:11-13, 53-82`.)*
- Cross-platform stubs: `composeApp/src/desktopMain/.../SandboxController.jvm.kt` and
  `composeApp/src/wasmJsMain/.../SandboxController.wasmJs.kt` are `NoOpSandboxController`
  implementations; the iOS source set ships an analogous no-op (the sandbox is **Android-only**
  by design). *(observed fact — `SandboxController.jvm.kt:6-29`; `SandboxController.wasmJs.kt:6-29`.)*
- Path-traversal protection: tar extraction in `RootfsDownloader.extractTar` checks
  `outFile.canonicalPath.startsWith(targetDir.canonicalPath)`. Path resolution in
  `SandboxFiles.resolveSandboxAbsolute` rejects `..` segments and verifies the resolved
  canonical path is rooted in the sandbox dir. *(observed fact — `RootfsDownloader.kt:124-128`,
  `SandboxFiles.kt:10-34`.)*

**System role:** Alpine + proot is the *tool-execution substrate* for the Android assistant.
Without it Kai is a chat client; with it Kai can install packages and run shell commands
on-device. Removing it on iOS/desktop/web is a deliberate scope cut, not a missing feature.
*(strong inference — README.md §Linux Sandbox + the no-op pattern.)*

**Portability hazards:** proot relies on Linux `ptrace`. iOS forbids `ptrace` and dynamic native
loading; desktop has the OS-native shell already; web has no equivalent. Any future port that
wants on-device tool execution needs a different primitive (e.g., Lima/Docker on macOS, WSL on
Windows, in-browser sandbox on web). *(portability hazard — sandbox is Android-only by design.)*

### Android foreground service (`DaemonService`)

**Where it lives:**
- Service class: `composeApp/src/androidMain/.../DaemonService.kt`. Channel ID
  `kai_daemon_channel`, notification ID 9001, importance LOW. `onCreate()` calls
  `startForeground()`; if it throws (Android 14+ time-limit / Android 12+ background-start),
  the service stops itself cleanly. `taskScheduler.start()` runs after the FGS upgrades. Returns
  `START_STICKY` from `onStartCommand`. Implements `onTimeout()` for Android 14+ FGS time
  limits — cleanly removes the notification and stops the service. *(observed fact —
  `DaemonService.kt:23-52`.)*
- Controller: `composeApp/src/androidMain/.../DaemonController.android.kt`. Calls
  `context.startForegroundService(Intent(...))` and catches
  `ForegroundServiceStartNotAllowedException` (silent — see contracts phase for behavior pinning).
  Auto-start gated by `appSettings.isDaemonEnabled()`. *(observed fact —
  `DaemonController.android.kt:18-30`.)*
- Activity re-assertion: `MainActivity.onStart()` calls `autoStartDaemon()` on every
  foreground transition. Comment explicitly attributes this to OEM-vendor battery managers
  (MIUI, EMUI/Huawei) silently killing the FGS while the activity is alive in the background.
  *(observed fact — `MainActivity.kt:91-107`.)*
- Manifest declarations: `AndroidManifest.xml` declares
  `<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>` and
  `<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC"/>`. The
  service element specifies `android:foregroundServiceType="dataSync"`. A second service
  (`com.inspiredandroid.kai.inference.ModelDownloadService`) shares the same FGS type for model
  downloads. *(observed fact — `androidApp/src/main/AndroidManifest.xml:9-10, 44-51`.)*
- Common-side abstraction: `expect fun createDaemonController(): DaemonController`; iOS / desktop /
  web get no-op implementations (no equivalent of FGS exists on those platforms). *(strong
  inference — same `expect/actual` pattern as the sandbox.)*

**Portability hazards:** FGS rules are Android-specific and changed materially across Android 12
(background-start ban), Android 13 (POST_NOTIFICATIONS runtime permission), and Android 14
(typed FGS + dataSync time limits). The `dataSync` time limit (currently 6 hours per
foreground-service run on Android 14+) is mitigated by `onTimeout()` + the `MainActivity.onStart`
re-assertion. *(portability hazard / observed fact — Android FGS API history; manifest
declarations.)*

### SAF (Storage Access Framework) binds

**Where it lives — and a deliberate absence:**
- **No direct SAF code in the repo.** A grep of androidMain for `ACTION_OPEN_DOCUMENT_TREE`,
  `takePersistableUriPermission`, `DocumentFile`, `OpenDocumentTree`, `StorageVolume`, etc.
  returned **no matches**. *(observed fact — exhaustive grep over `**/*.kt` and `**/*.xml`.)*
- **Indirect SAF via FileKit:** `MainActivity.onCreate()` calls `FileKit.init(this)`
  (`io.github.vinceglb.filekit`). FileKit handles SAF (or its analogue on each platform) for
  file pickers and file save dialogs internally; Kai consumes its cross-platform API and never
  touches `Intent.ACTION_OPEN_DOCUMENT*` directly. *(observed fact — `MainActivity.kt:25, 36`;
  multiple `commonMain` ViewModels import `io.github.vinceglb.filekit`.)*
- **FileProvider for outbound sharing:** `androidApp/src/main/AndroidManifest.xml:35-43`
  declares `androidx.core.content.FileProvider` with authority `${applicationId}.fileprovider`
  and a `<meta-data>` reference to `@xml/file_paths`. The sandbox uses this in
  `composeApp/src/androidMain/.../sandbox/SandboxFiles.kt`'s `openFileWithIntent()`:
  `FileProvider.getUriForFile(context, "${packageName}.fileprovider", file)` followed by
  `Intent.ACTION_VIEW` so a sandbox-produced file can be opened in another app
  (image viewer, text editor, etc.). *(observed fact — `SandboxFiles.kt:50-72`.)*
- **Sandbox `/root` placement is the SAF-adjacent design choice:** `LinuxSandboxManager.homePath`
  returns `getExternalFilesDir(null)/sandbox-home`, which lives under app-private external
  storage (Android's "scoped storage" model) — visible to FileProvider, not visible to other
  apps. *(observed fact — `LinuxSandboxManager.kt:43-58`.)*

**System role:** SAF is **delegated to FileKit** for pickers and **avoided entirely** for
durable URI grants. Files Kai sources externally are read at picker time; files Kai produces are
shared *outward* via the app's own FileProvider. There is no persistent SAF-grant management,
no `takePersistableUriPermission` cache. *(strong inference — exhaustive negative grep + the
positive FileKit usage.)*

**Portability hazards:** FileKit's API is consistent across platforms but the back-end
permission model differs (SAF on Android, NSURL bookmark on iOS, `<input type=file>` on web,
native chooser on desktop). Persistent re-access to user-chosen files across app restarts is
*not* implemented today; if a future feature needs that, SAF-direct (or a FileKit upgrade) will
be required. *(portability hazard / open question.)*

### Battery whitelist / Doze / power management

**Where it lives — and another deliberate absence:**
- **No `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` request anywhere in the repo.** A grep over
  `**/*.kt` and `**/*.xml` for `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`,
  `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`, `isIgnoringBatteryOptimizations`, and
  `PowerManager` returned **no matches**. The manifest does not request the
  `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` permission. *(observed fact — exhaustive grep.)*
- **Mitigation strategy is deliberately at the FGS layer:** the daemon uses
  `START_STICKY` so the OS re-creates it after a kill; `MainActivity.onStart()` re-asserts
  `startForegroundService()` on every foreground (idempotent if already up); `onTimeout()`
  handles Android 14+ FGS time limits. The code comment at `MainActivity.kt:91-100` explicitly
  attributes the re-assertion to "aggressive OEM battery managers (MIUI, EMUI/Huawei)" rather
  than the standard Doze / App Standby pipeline. *(observed fact — `MainActivity.kt:91-107`,
  `DaemonService.kt:40, 44`.)*
- **Implication for the user:** under stock Android Doze + standard App Standby, Kai's daemon
  *should* survive while the app is bound to the foreground notification. Under aggressive OEM
  power management (Xiaomi MIUI, Huawei EMUI, OPPO ColorOS), the daemon may be killed despite
  the foreground notification; the user must reopen the app for scheduling to resume — at which
  point `onStart()` re-asserts. *(strong inference — code comment + the absence of any battery
  whitelist code.)*

**System role:** Kai chooses **not to ask** for battery whitelist. The product trade-off is
"prefer fewer permission prompts; rely on FGS + re-assertion." The contracts phase will pin the
user-visible failure mode (heartbeat misses; user-invisible until next foreground).
*(strong inference — explicit absence of a request flow + the comment in MainActivity.)*

**Portability hazards:** Doze and OEM battery managers are Android-specific; iOS' background
execution model is much stricter (no equivalent FGS); desktop / web have no Doze analogue.
Any iOS / desktop port of "autonomous heartbeat" must re-design the wakeup model
(BGTaskScheduler on iOS, `JobScheduler`/cron on desktop, server-pushed on web) — the Android
strategy does not transfer. *(portability hazard.)*

---

## Durable State

| State | Location | Notes |
|---|---|---|
| Encrypted conversations | local Android storage (per README.md) | "Encrypted storage" feature; conversation ids tie to sandbox session ids when persistable. |
| Persistent memory entries | local app storage | Surfaced via the "Persistent memory" feature. |
| App settings | per-platform settings store (DataStore on Android, NSUserDefaults on iOS, file on desktop, localStorage on wasmJs — typical KMP shape) | Includes daemon-enabled flag (`AppSettings.isDaemonEnabled()`). |
| Sandbox rootfs | `context.filesDir/linux-sandbox/rootfs/` | Alpine extraction. |
| Sandbox `/root` | `context.getExternalFilesDir(null)/sandbox-home` | Bind-mounted into the sandbox at `/root` so files are visible to FileProvider. |
| Sandbox tmp | `context.filesDir/linux-sandbox/tmp/` | Sandbox's `/tmp`. |
| Native libs (proot, talloc) | `applicationInfo.nativeLibraryDir/lib*.so` | Extracted from APK by Android at install. |
| Per-session shell transcripts | in-memory `SnapshotStateList<TerminalLine>` per `sessionId`; persisted only for chat-bound sessions | `SandboxSessions.isPersistable` controls persistence. |
| Heartbeat conversation state | local app storage; surfaced via deep-link `EXTRA_OPEN_HEARTBEAT` notification tap | One pending heartbeat conversation at a time (notification ID 9002 fixed). |
| Settings export/import | JSON file the user picks via FileKit | Backup format. |
| LiteRT model files | local app storage | Downloaded by `ModelDownloadService` (separate FGS). |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ1 | needs-runtime-test | Does `LinuxSandboxManager.makeWritable()` recurse symlinks? Source comment says "walkTopDown" — would need a runtime test on a rootfs that contains a symlink-to-directory to confirm the walk policy. | Not load-bearing for the architecture map; relevant to defect-scan-mechanical if that pipeline runs later. |
| arch-OQ2 | needs-maintainer-decision | The `:screenshotTests` module is excluded from default Gradle assemble — is it CI-only or also runnable locally? | Not load-bearing for porting; relevant only if a port wants to mirror the test surface. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| kai-CF-alpine-proot | contracts | Alpine/proot bootstrap — pin user-visible setup contract (success/failure surface, retry semantics, recoverable vs unrecoverable error). Architecture has located the code (this document §Focus subsystem deep-dives) and characterised the state machine. | User-facing behavior is the contracts phase's rubric. |
| kai-CF-foreground-service | contracts | Foreground daemon — pin notification visibility, OS-kill recovery contract, behavior under FGS time limits. Architecture has located the service and re-assertion code. | Lifecycle contract is observable user-facing behavior. |
| kai-CF-saf-binds | contracts | SAF binds — pin permission flow (delegated to FileKit), the FileProvider-based outbound sharing contract, and the **deliberate absence** of persistent SAF grants. Architecture has confirmed there's no direct SAF code. | Permission-flow + persistability semantics are user-visible contracts. |
| kai-CF-battery-whitelist | contracts | Battery whitelist — pin the explicit non-request decision and the resulting user-visible behavior under Doze and aggressive OEM power management. Architecture has confirmed the absence of any battery-whitelist request and located the FGS-layer mitigation. | Request flow + fallback behavior are user-visible contracts. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system intent is documented. | PASS | §System Intent (paragraph). |
| 2 | The layer map and dependency direction are documented. | PASS | §Layer Map (Package Inventory table, Dependency Direction). |
| 3 | Public surfaces are identified. | PASS | §Public Surfaces (Android user-facing, iOS/Desktop/Web user-facing, external integrations consumed, wire formats). |
| 4 | Runtime lifecycle, concurrency model, and porting priorities are summarized. | PASS | §Runtime Lifecycle, §Concurrency Model, §Porting Priorities (table). |
| 5 | Findings are marked with evidence levels. | PASS | Every claim tagged inline with *observed fact*, *strong inference*, or *portability hazard*. Two open questions captured in §Open Questions. |

**Validated by:** 2026-05-06 (architecture phase, session 1)
**Overall:** PASS
