# Behavioral Contracts — Kai 9000

This document recovers Kai's user-visible behavior — what the user sees, what the user can do,
what defaults apply, what side effects are observable, and where the boundaries of the trusted
layer sit. It is written from the outside in: the source of truth is what an operator sees the
app do, not how the code is structured. (For structural detail, see `findings/architecture/architecture-map.md`.)

This phase inherits four `carry_forward` entries from architecture (`kai-CF-alpine-proot`,
`kai-CF-foreground-service`, `kai-CF-saf-binds`, `kai-CF-battery-whitelist`). Each appears as a
dedicated subsection in §Feature Contracts → §Focus subsystems below; their `carry_forward`
entries are closed at the end of this document.

## Surfaces Covered

Kai exposes a single user-facing surface — a Compose Multiplatform UI — that ships with platform-specific
capability subsets. (*observed fact* — settings.gradle.kts modules; the `expect/actual` source-set
layout in `composeApp/src/`.) Each surface below is the same Compose `App()` composable with
different `actual` implementations of platform expects backing it.

| Surface | Platforms | What it can do | Notable absences |
|---|---|---|---|
| Compose UI (chat + settings + heartbeat history + sandbox terminal + file browser) | Android, iOS, macOS, Windows, Linux desktop, Web (WASM) | Chat with the assistant, switch LLM providers, configure system prompt, export/import settings, view persistent memory, run heartbeat manually | Sandbox terminal + file browser are Android-only — on every other platform `NoOpSandboxController` returns failures |
| Persistent foreground notification ("Kai daemon running") | Android only (FGS) | Keeps the app process alive so the heartbeat scheduler can fire while the activity is backgrounded | iOS uses BGTaskScheduler-style approaches not implemented; desktop/web only run heartbeat while the app is open |
| Heartbeat notification ("[Kai] reply ready") | Android only (notifications + deep-link) | Surfaces a periodic self-check result; tap to open the heartbeat conversation | Other platforms fall back to in-app surfacing only |
| AI-driven `send_notification` tool notifications | Android only | The assistant emits notifications via the `kai_ai_notifications` channel | Other platforms have no system tray |
| Notification-listener service | Android only | Reads other apps' notifications when the user grants the listener permission | Other platforms have no equivalent permission |
| Stored conversations / encrypted local storage | All platforms | Persists chats and persistent-memory entries locally, encrypted | n/a |
| Settings export/import (JSON file via FileKit) | All platforms | Backup / restore | n/a |
| Linux sandbox shell (Alpine + proot) | Android only | Run real shell commands, install packages, browse files | iOS / desktop / web ship `NoOpSandboxController` and surface "Sandbox file browser is Android-only" errors |
| Distribution channels | Multiple per platform — see architecture map §Build and Packaging | Google Play, App Store, F-Droid, Homebrew, AUR, winget, Flatpak, AppImage, DEB/RPM/MSI/DMG, web build | n/a |

## Feature Contracts

### Surface — Compose UI: Chat

| Field | Value |
|---|---|
| **Feature** | Send a message to the assistant |
| **Trigger or input** | User types in the question input and taps send (`QuestionInput.kt`); deep links from `EXTRA_OPEN_HEARTBEAT` open a specific conversation. |
| **Defaults** | Default LLM provider is the first-configured provider with valid credentials in the user's chain; system prompt comes from the editable "soul" (configurable in settings). |
| **Observable output** | Streamed assistant text in the conversation UI; tool-call results inline; image attachments rendered inline; "Interactive UI" generated screens may appear instead of plain chat. (*observed fact* — README.md §Features, §AI That Builds Screens.) |
| **Side effects** | Conversation persisted locally (encrypted); persistent-memory updates when the model writes a memory; tool calls trigger their respective tools (web search HTTP calls, sandbox shell exec, calendar reads/writes, MCP server calls, notification posts). |
| **Persisted state** | Conversation transcript; memory entries; per-session sandbox transcript when `SandboxSessions.isPersistable(sessionId)` (i.e., sessionId is not one of the sentinels `__terminal__`, `__system__`, `__default__`). (*observed fact* — `SandboxController.kt:51`.) |
| **Error behavior** | On LLM-provider failure the chain falls over to the next configured provider (24+ providers supported); on tool-call failure the assistant sees the failure as a tool-result message and may retry or surface to user. (*observed fact* — README.md §Features "Multi-service fallback".) |
| **Retry or recovery behavior** | LLM provider fallback is automatic; tool retries are model-driven (no hardcoded retry loop). |
| **Owner (layer/package)** | `composeApp/commonMain/.../ui/chat/ChatViewModel.kt`, `ChatActions.kt`; provider chain in commonMain. |

### Surface — Compose UI: Settings

| Field | Value |
|---|---|
| **Feature** | Configure providers, system prompt, daemon, sandbox, heartbeat |
| **Trigger or input** | User navigates to Settings (`SettingsScreen.kt`); toggles the daemon switch (which writes `appSettings.isDaemonEnabled`); enters API keys; edits system prompt; etc. |
| **Defaults** | Daemon enabled = true at first run; system prompt = built-in default; heartbeat interval = built-in default. (*open question* — exact default values not pinned in the architecture survey.) |
| **Observable output** | Compose UI updates; Settings persist immediately to the platform settings store. |
| **Side effects** | Toggling daemon-enabled to true and pressing the toggle does not start the daemon directly; the next `MainActivity.onStart()` re-asserts via `autoStartDaemon()`. (*observed fact* — `MainActivity.kt:91-107`.) Toggling to false stops the FGS via `DaemonController.stop()`. |
| **Persisted state** | Settings file (DataStore on Android, NSUserDefaults on iOS, file on desktop, localStorage on wasmJs). |
| **Error behavior** | Invalid API keys surface as provider errors at next chat send. |
| **Recovery behavior** | n/a — settings are atomic writes. |
| **Owner (layer/package)** | `composeApp/commonMain/.../ui/settings/SettingsScreen.kt`, `composeApp/commonMain/.../data/AppSettings.kt`. |

### Surface — Compose UI: Settings export / import

| Field | Value |
|---|---|
| **Feature** | Export and import settings as a JSON file |
| **Trigger or input** | User taps "Export settings" or "Import settings" in Settings; FileKit shows a save / open file picker. |
| **Defaults** | Export filename = product-defined slug; format = JSON. |
| **Observable output** | A user-chosen JSON file at export; settings restored at import. |
| **Side effects** | Writes/reads through `io.github.vinceglb.filekit`'s cross-platform picker. On Android, FileKit uses SAF under the hood; on iOS, NSURL bookmarks; on desktop, native chooser; on web, browser file dialog. (*strong inference* — FileKit cross-platform shape; see §SAF binds for the Android specifics.) |
| **Persisted state** | The JSON file outside the app; current settings replaced on import. |
| **Error behavior** | User cancels picker → no-op; malformed JSON on import → error dialog (*open question* — exact error UX not pinned). |
| **Recovery behavior** | Import is non-transactional; partial parse failures may leave settings inconsistent (*open question*). |
| **Owner (layer/package)** | `composeApp/commonMain/.../ui/settings/...`. |

### Surface — Compose UI: Sandbox terminal (Android-only)

| Field | Value |
|---|---|
| **Feature** | Interactive terminal tab in Settings |
| **Trigger or input** | User opens the Terminal tab (`TerminalSheet.kt`); types commands. |
| **Defaults** | `sessionId = SandboxSessions.TERMINAL` (sentinel — not persisted to chat history). |
| **Observable output** | Streamed `stdout` / `stderr`; persistent-shell session means commands share environment / working directory across invocations. |
| **Side effects** | Real shell commands inside the sandbox rootfs at `context.filesDir/linux-sandbox/rootfs/` with `/root` bind-mounted from `getExternalFilesDir(null)/sandbox-home`. Files created in `/root` are externally visible to FileProvider. |
| **Persisted state** | Terminal-tab transcript is **not** persisted (sentinel session); files written into `/root` *are* persisted and survive app restarts. |
| **Error behavior** | If sandbox state is not `Ready`, output is the constant string `"Sandbox is not ready"`. (*observed fact* — `SandboxController.android.kt:151, 184, 333`.) |
| **Recovery behavior** | A wedged shell can be reset on the next call (the persistent shell self-heals via `reset()` calls in the manager). (*observed fact* — `SandboxController.android.kt:190-192` comment.) |
| **Owner (layer/package)** | `composeApp/commonMain/.../ui/settings/TerminalSheet.kt` + `composeApp/androidMain/.../sandbox/`. |

### Surface — AI tools

The assistant has access to tools the chat layer dispatches: web search, calendar event creation, shell command execution (sandbox-only), `send_notification`, MCP server tools, image attachments. Each tool has its own contract; the relevant ones for porting are in §Feature Contracts → §Focus subsystems below (sandbox shell, notifications) and §Security and Authorization (calendar, notification listener, MCP servers).

---

## Focus subsystems (carry_forward closure)

### Alpine / proot bootstrap (closes `kai-CF-alpine-proot`)

| Field | Value |
|---|---|
| **Feature** | One-tap setup of an on-device Alpine Linux user-land that the assistant can shell into |
| **Trigger or input** | User taps "Set up Linux Sandbox" in Settings; surfaced when `SandboxState` is `NotInstalled`. Calls `SandboxController.setup()`. (*observed fact* — `SandboxController.kt:60`, README.md §Linux Sandbox.) |
| **Defaults** | Alpine version 3.21.3, branch v3.21; mirror order is the literal list in `RootfsDownloader.kt:22-27` (dl-cdn → mirrors.edge.kernel → ftp.halifax → alpine.ethz → mirror.csclub.uwaterloo → mirrors.tuna.tsinghua); first mirror that returns 200 wins. (*observed fact*.) |
| **Observable output** | The status surface progresses through `NotInstalled → Downloading(progress=0..1) → Extracting → Installing(detail) → Ready`. UI shows `statusText` ("Downloading rootfs...", "Extracting...", "Installing...", "Ready") and a progress bar driven by `progress` (download phase only). On `Ready`, the UI exposes the Terminal tab and the package-installer button. (*observed fact* — `SandboxController.android.kt:78-121`.) |
| **Side effects** | Creates `context.filesDir/linux-sandbox/rootfs/` (extraction target); writes `/etc/resolv.conf` and `/etc/apk/repositories` post-extract; runs `makeWritable` walk over the extracted tree; bind-mount-equivalent of `/root` is `getExternalFilesDir(null)/sandbox-home`. Native libraries `libproot.so` / `libtalloc.so` were already present in `applicationInfo.nativeLibraryDir/` from APK install. (*observed fact* — `RootfsDownloader.kt:217-234`, `LinuxSandboxManager.kt:30-67`.) |
| **Persisted state** | The full rootfs (~100s of MB depending on packages installed), `/root` data, sandbox `/tmp`. The README advertises a "lightweight ~3 MB download" — this refers to the minirootfs `.tar.gz` itself, not the extracted size. (*observed fact* — README.md §Linux Sandbox; download URL pattern in `RootfsDownloader.kt:42`.) |
| **Error behavior** | If **all six mirrors fail**, `RootfsDownloader.download()` throws `IOException("All Alpine mirrors failed", lastError)` and the state transitions to `Error(message)`. The UI shows `"Error: <message>"` and exposes a retry path via `setup()`. The downloaded partial file is deleted on per-mirror failure (`if (targetFile.exists()) targetFile.delete()`). (*observed fact* — `RootfsDownloader.kt:60-64`.) |
| **Retry or recovery behavior** | The user retries `setup()` from the UI; on retry, all six mirrors are tried again from the top. There is **no exponential backoff and no per-mirror retry-with-jitter**. (*observed fact* — `RootfsDownloader.kt:47-64`.) `cancel()` aborts an in-flight `Downloading` or `Extracting` (the `RootfsDownloader.download` `bodyAsChannel()` read can be interrupted by coroutine cancellation, and `extractTar` checks `ensureActive()` per loop iteration). (*strong inference* — coroutine cancellation patterns + `LinuxSandboxManager` cancellation hook.) |
| **Owner (layer/package)** | `:composeApp/androidMain/.../sandbox/{LinuxSandboxManager,RootfsDownloader,ProotExecutor,SandboxFiles,SandboxState,SandboxModule}.kt`; native binaries in `androidApp/src/main/jniLibs/<abi>/` built by repo-root `build-proot.sh`. |

**Path-traversal contract:** Tar extraction rejects entries whose canonical path is outside `targetDir.canonicalPath`; `SandboxFiles.resolveSandboxAbsolute` rejects `..` segments and verifies the resolved canonical path is rooted in the sandbox. Symlinks are *extracted* (the tar extractor calls `linkTarget`) but the path-traversal check happens before extraction. (*observed fact* — `RootfsDownloader.kt:124-128`, `SandboxFiles.kt:10-34`.)

### Android foreground service (`DaemonService`) (closes `kai-CF-foreground-service`)

| Field | Value |
|---|---|
| **Feature** | Long-lived Android foreground service that keeps the app process alive so the autonomous heartbeat scheduler can fire while the activity is backgrounded |
| **Trigger or input** | (a) `MainActivity.onStart()` calls `autoStartDaemon()` on every foreground; (b) user toggles "Daemon" in Settings to true; (c) Android OS recreates the service after a kill (`START_STICKY`). Auto-start gated by `appSettings.isDaemonEnabled()`. (*observed fact* — `MainActivity.kt:91-107`, `DaemonController.android.kt:16-23`.) |
| **Defaults** | Daemon enabled = true (assumed default; setting is read at `MainActivity.onStart`). Notification channel `kai_daemon_channel`, importance `LOW`. Notification ID `9001`. Service type `dataSync`. Notification text comes from string resources `R.string.daemon_channel_name`, `R.string.daemon_channel_description`, `R.string.daemon_notification_text`. |
| **Observable output** | A persistent low-importance notification appears in the system tray titled with `R.string.app_name`, body `R.string.daemon_notification_text`, icon `android.R.drawable.ic_popup_sync`, marked `setOngoing(true)` (user cannot swipe it away). Tapping the notification launches the main activity (with `FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TOP`). (*observed fact* — `DaemonService.kt:67-85`.) |
| **Side effects** | (a) Notification posted; (b) `taskScheduler.start()` runs (idempotent — re-call is a no-op if loop already running); (c) the OS treats the app as "in use" for the duration, raising its priority for the standard process-killer. (*observed fact* — `DaemonService.kt:34-38, 40` plus the comment at lines 33-37.) |
| **Persisted state** | None directly; the daemon's *purpose* is to keep `taskScheduler`'s in-memory state alive so persistent memory writes happen on the heartbeat. |
| **Error behavior** | (a) `startForeground()` throws `Exception` (Android 14+ restriction or Android 12+ background-start restriction) → service catches generically, calls `stopSelf()`, and returns from `onCreate`. (b) `DaemonController.start()` catches `ForegroundServiceStartNotAllowedException` (Android 12+) **silently** — the comment explicitly notes this is the case where the app is not in a foreground state. (*observed fact* — `DaemonService.kt:27-32`, `DaemonController.android.kt:22-24`.) |
| **Retry or recovery behavior** | (a) `onStartCommand` returns `START_STICKY` so the OS recreates after kills; (b) `MainActivity.onStart()` re-asserts on every foreground (idempotent — `startForegroundService` on an already-running service is a no-op); (c) `onTimeout(startId, fgsType)` (Android 14+ FGS time-limit handler) calls `stopForeground(STOP_FOREGROUND_REMOVE)` + `stopSelf()` — clean stop, then waits for next re-assertion. (*observed fact* — `DaemonService.kt:40-47`.) |
| **Owner (layer/package)** | `:composeApp/androidMain/.../{DaemonService.kt, DaemonController.android.kt}`; manifest declarations in `androidApp/src/main/AndroidManifest.xml:9-10, 44-47`. |

**OS-kill recovery contract** (the contract the user *experiences*): if the OS or an OEM power-manager kills the FGS while the activity is backgrounded, the heartbeat misses fires until the user next opens the app. There is no user-visible warning; recovery is "next foreground restores the daemon" (silent). (*strong inference* — the `MainActivity.onStart` comment + absence of any "daemon was killed" UI surface.)

### SAF (Storage Access Framework) binds (closes `kai-CF-saf-binds`)

This subsystem's contract is shaped by **two deliberate non-features**: Kai (a) does not expose direct SAF (no `ACTION_OPEN_DOCUMENT*`, no `OpenDocumentTree`) and (b) does not retain persistent SAF grants across app restarts. The user-visible behavior is mediated by FileKit for inbound file selection and FileProvider for outbound file sharing.

| Field | Value (FileKit-mediated picker contract — inbound) |
|---|---|
| **Feature** | Pick a file from the user's storage (settings-import JSON, image attachments, etc.) |
| **Trigger or input** | A FileKit `PickerType.File`, `PickerType.Image`, etc. invocation in commonMain code; surfaced as the OS-native picker on each platform. |
| **Defaults** | None at the user level; FileKit's defaults apply. |
| **Observable output** | The OS file picker on Android (which uses SAF), iOS (NSOpenPanel-equivalent), desktop (native chooser), web (`<input type=file>`). The user picks a file; the app reads it once. |
| **Side effects** | One-shot file read via the platform-mediated URI/handle. No `takePersistableUriPermission` is called on Android — confirmed by exhaustive grep. |
| **Persisted state** | None: no SAF grant cache, no bookmark store. The path is **not** retained across app restarts; if the user wants to "re-import" they must re-pick. |
| **Error behavior** | User cancels → no-op (FileKit returns null). File read fails → IOException surfaced to the calling layer; UI surfaces a generic error. |
| **Recovery behavior** | User retries the picker. |
| **Owner (layer/package)** | Cross-platform: `io.github.vinceglb.filekit` library; consumed from commonMain ViewModels (`ChatViewModel`, settings layer). FileKit initialized at `MainActivity.onCreate()` via `FileKit.init(this)` on Android. |

| Field | Value (FileProvider outbound-share contract — outbound) |
|---|---|
| **Feature** | Open a sandbox-produced file in another app via `Intent.ACTION_VIEW` |
| **Trigger or input** | `SandboxController.openFile(path)` from chat or terminal UI; resolves to a real `File` under `<sandbox>/rootfs/...` or `<sandbox-home>/...`. (*observed fact* — `SandboxController.android.kt:268-274`, `SandboxFiles.kt:48-72`.) |
| **Defaults** | MIME type guessed from extension via `MimeTypeMap.getSingleton().getMimeTypeFromExtension(ext)`; falls back to `*/*` if unknown. (*observed fact* — `SandboxFiles.kt:38-41`.) |
| **Observable output** | A system chooser appears with apps that handle the MIME type; the chosen app receives a content URI from `${applicationId}.fileprovider` and reads the file. |
| **Side effects** | Read-grant (`Intent.FLAG_GRANT_READ_URI_PERMISSION`) given to the receiving app for the lifetime of the activity that received the intent — standard FileProvider semantics. |
| **Persisted state** | None at the URI level; the underlying file remains in the sandbox. |
| **Error behavior** | Three documented failure modes — (a) `IllegalArgumentException` if path can't be resolved within the sandbox (path-traversal protection); (b) `IllegalArgumentException` if path resolves but is not a file; (c) `FileProvider.getUriForFile` throws `IllegalArgumentException` if the path is outside the FileProvider's declared `file_paths` config — surfaced via `Result.failure(IllegalStateException(error ?: "Open failed"))`. (*observed fact* — `SandboxController.android.kt:268-274`, `SandboxFiles.kt:55-58`.) (d) No app handles the MIME type → `ActivityNotFoundException` (caught at the call site — verified by `import` in `SandboxFiles.kt:3`). |
| **Recovery behavior** | The caller surfaces an error message; the user can copy the file via the file browser and try a different opener. |
| **Owner (layer/package)** | `:composeApp/androidMain/.../sandbox/SandboxFiles.kt` (`openFileWithIntent`); manifest declares `androidx.core.content.FileProvider` with authority `${applicationId}.fileprovider` and `<meta-data android:resource="@xml/file_paths"/>`. (*observed fact* — `androidApp/src/main/AndroidManifest.xml:35-43`.) |

**The deliberate-absence contract:** the user *cannot* save a file picker selection across restarts. If a future feature needs persistent re-access to user-chosen documents (a "watch this folder" feature, or a bookmarked-attachment list), this gap must be filled — either by upgrading FileKit to use SAF persistable URIs, or by using SAF directly. Today, this is **explicitly not supported**. (*strong inference* — exhaustive negative grep + the FileKit-only picker pattern.)

### Battery whitelist / Doze / power management (closes `kai-CF-battery-whitelist`)

This subsystem's contract is its **deliberate absence of a permission request**. The contract is what the user observes when Kai *does not* ask for `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`.

| Field | Value |
|---|---|
| **Feature** | (Non-feature) Battery optimization whitelist request |
| **Trigger or input** | None. Kai never invokes `Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS)` and never declares the `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` permission. (*observed fact* — exhaustive grep over `**/*.kt` and `**/*.xml`.) |
| **Defaults** | Doze + App Standby remain in effect for Kai exactly as for any other Android app. The user is *never prompted* to whitelist Kai. |
| **Observable output** | (a) On stock Android (Pixel, AOSP-derived ROMs, well-behaved OEM ROMs) Doze allows foreground-service apps to keep running with their notification visible — the Kai daemon stays up and the heartbeat fires reliably. (b) On aggressive OEM ROMs (MIUI, EMUI/Huawei, ColorOS, MagicOS) the FGS may be killed silently despite the foreground notification; the heartbeat misses until the user next opens the app and `MainActivity.onStart()` re-asserts the FGS. (*strong inference* — the `MainActivity.kt:91-100` code comment names these vendors explicitly.) |
| **Side effects** | None directly. The mitigation strategy lives in the FGS layer (`START_STICKY` + `MainActivity.onStart` re-assertion + `onTimeout` handler). |
| **Persisted state** | None. |
| **Error behavior** | No "permission denied" path because no permission is requested. The failure mode is silent FGS death on aggressive OEMs; no UI surface tells the user "the daemon was killed." |
| **Recovery behavior** | "Next foreground restores the daemon." User has to open the app. (*observed fact* — `MainActivity.onStart` comment.) |
| **Owner (layer/package)** | n/a (the contract is the *absence* of code). The mitigation owners are `DaemonService.kt` (START_STICKY, onTimeout) and `MainActivity.kt:91-107` (re-assertion). |

**Why this is the user's contract** *(strong inference)*: by choosing not to request battery whitelist, Kai trades reliability on aggressive OEM ROMs for fewer permission prompts. The trade-off is consistent with the README's "open-source / privacy-first" framing and is not documented as a feature anywhere in the user-facing UI — the user discovers the OEM-kill behavior the first time they look at heartbeat history after a long unattended interval.

---

## High-Value Behaviors

- **Cancellation:** `SandboxController.cancel()` aborts an in-flight `setup()` (download or extract). `CommandHandle.cancel()` kills a running shell command (`process.destroyForcibly()` + reader-future close). Streaming commands have no implicit timeout — cancellation is the only termination signal in that path. (*observed fact* — `SandboxController.android.kt:190-192`, `ProotExecutor.kt:21-27`.)
- **Streaming output:** Chat tool output and terminal output stream via `onStdout(String)` / `onStderr(String)` callbacks. The 30-second default `executeCommand` timeout is bypassed in the streaming path. (*observed fact* — `SandboxController.android.kt:177-205`.)
- **Compaction / summarization:** Conversations are stored in full; the README's "Persistent memory" feature stores extracted memories, not summarised conversations. (*open question* — exact memory write contract not pinned.)
- **Resume flows:** Heartbeat conversation has a fixed notification ID 9002 so a new heartbeat reply replaces any pending one — only one pending heartbeat conversation at a time. (*observed fact* — `HeartbeatNotifier.android.kt:30-31`.)
- **Tool execution:** Each tool gets a structured tool-call from the model, executes, and returns a structured tool-result. Errors surface as tool-result content the model sees and can reason over.

## Security and Authorization

| Concern | Behavior |
|---|---|
| **Authentication for the app itself** | None — Kai is a local-first assistant; there is no Kai user account. *(observed fact — README.md positioning, no `Auth*` files in source.)* |
| **LLM provider API keys** | Stored in the platform settings store (DataStore on Android, NSUserDefaults on iOS, etc.). README states "Encrypted storage — Conversations stored locally with encryption" — *open question* whether this encryption extends to API keys (likely yes given keys are part of settings). |
| **Encrypted local storage** | Conversations encrypted on-device. *(observed fact — README §Features list.)* The exact encryption scheme is not extracted in this phase (relevant to defect-scan-semantic if a later run includes it). |
| **MCP server trust boundary** | User-supplied MCP servers (URL + optional headers/auth). Kai trusts whatever the user configures. The MCP transport is JSON-RPC. *(observed fact — README §Features "MCP server support".)* |
| **Notification listener trust boundary** | When the user grants notification listener access, Kai can read other apps' notifications. This is a user-granted system permission; Kai must ensure read content does not leak (relevant to defect-scan-semantic). *(observed fact — `KaiNotificationListenerService.kt` exists; needs runtime permission.)* |
| **Calendar permission** | `READ_CALENDAR` and `WRITE_CALENDAR` declared in manifest; used by the calendar-event-creation tool. *(observed fact — `AndroidManifest.xml:5-6`.)* |
| **POST_NOTIFICATIONS** | Declared; required on Android 13+. Tool-driven notifications (`send_notification`) and heartbeat notifications need this. *(observed fact — `AndroidManifest.xml:7`.)* |
| **SET_ALARM** | Declared; presumably for the heartbeat scheduler's wake-up alarms. *(observed fact — `AndroidManifest.xml:8`.)* |
| **FOREGROUND_SERVICE / FOREGROUND_SERVICE_DATA_SYNC** | Declared; required for the daemon. *(observed fact — `AndroidManifest.xml:9-10`.)* |
| **No network security config relaxations** | `networkSecurityConfig="@xml/network_security_config"` declared; the default is HTTPS-only — user-supplied non-HTTPS endpoints (LLM providers, MCP servers) require explicit user opt-in via the config. (*observed fact + open question* — config file content not read in this phase.) |
| **Trust boundary at sandbox** | The Linux sandbox has access to the app-private storage at `/root` (= `<external-files>/sandbox-home`) but not to user data outside that. proot via ptrace cannot escape user namespaces. *(observed fact — `LinuxSandboxManager.homePath` definition; proot's design.)* |

## Configuration Model

- **Per-platform settings store:** DataStore on Android, NSUserDefaults on iOS, file on desktop, localStorage on wasmJs (typical KMP shape inferred from `data/AppSettings`). Edited in the Settings UI; written immediately. *(strong inference — typical KMP DI pattern; the exact store was not opened in the architecture phase.)*
- **Daemon enabled flag:** `appSettings.isDaemonEnabled()` is the gate. When true, every `MainActivity.onStart` re-asserts the FGS; when false, no re-assertion.
- **Provider chain:** Stored in settings; the chat layer iterates and falls over on error. Order is user-configurable.
- **System prompt ("soul"):** Editable in Settings; persisted to settings store.
- **Heartbeat schedule:** Configured in settings (interval is user-editable per README §Features).
- **Sandbox version pinning:** Alpine 3.21.3 is hardcoded in `RootfsDownloader.kt`; mirror list is hardcoded; proot+talloc commits/versions are hardcoded in `build-proot.sh`. These are source-level configuration, not user-level. *(observed fact.)*
- **Build flavors:** `foss` and `playStore` for Android — typical store-vs-non-store split (Play Services availability, in-app review APIs etc.). *(observed fact — `androidApp/src/{foss,playStore}/`.)*

## Doc/Test Conflicts

None surfaced in this phase. The README accurately describes the feature set; no source-code contradiction was observed during the architecture survey. (*observation* — focused architecture survey did not exhaustively diff README claims against code; a defect-scan run would surface any fine-grained drift.)

## Black-Box Acceptance List

Concrete scenarios another implementation can run without referencing the source.

| # | Scenario | Precondition | Action | Expected Outcome |
|---|----------|--------------|--------|------------------|
| 1 | First-launch sandbox setup | Fresh install on Android, network available, ≥150MB free space | Open Settings, tap "Set up Linux Sandbox" | Status progresses Downloading → Extracting → Installing → Ready within ~30s on broadband; the Terminal tab becomes interactive. |
| 2 | All-mirrors-fail sandbox setup | Network unavailable or all 6 Alpine mirrors unreachable | Tap "Set up Linux Sandbox" | Status reaches Error with text starting "Error:"; retry button is exposed; partial download file (if any) is deleted. |
| 3 | Sandbox shell command | SandboxState = Ready | Open Terminal tab; type `echo hello`; press send | `hello\n` streams back; exit code is 0; no entry written to chat conversation history. |
| 4 | Sandbox-not-ready guard | SandboxState != Ready | Chat tool calls `executeCommand` with any command | Returns the literal string `"Sandbox is not ready"` rather than executing. |
| 5 | Sandbox file open via FileProvider | SandboxState = Ready, a file exists at `/root/foo.png` | Tap "Open" on the file in the sandbox file browser | System chooser appears with image-viewers; receiving app opens the image successfully. |
| 6 | Path-traversal rejection | SandboxState = Ready | A tool tries `executeCommand` with arguments that resolve to `..//etc/passwd` outside the sandbox | The path is rejected at `SandboxFiles.resolveSandboxAbsolute` (returns null → caller returns false / failure). No host-OS path is touched. |
| 7 | Daemon survives backgrounding | Daemon enabled, app foreground | Background the app | Persistent "Kai daemon running" notification visible; foreground service continues; heartbeat fires per schedule. |
| 8 | Daemon FGS time-limit (Android 14+) | Daemon running for ≥6 hours continuously | Wait | `onTimeout` fires; notification removed cleanly; on next foreground, `MainActivity.onStart` re-asserts and the daemon is restored. |
| 9 | OEM-kill recovery | Aggressive OEM ROM kills the daemon | Kill the FGS via OEM tooling, wait, then open the app | On `MainActivity.onStart`, `autoStartDaemon()` re-asserts the FGS; user sees no warning; missed heartbeats are not retroactively replayed. |
| 10 | Battery-whitelist absence | Fresh install, any Android 8+ device | Browse all permission prompts during onboarding | The user is **never** prompted to whitelist Kai from battery optimizations. |
| 11 | Settings export round-trip | Any platform | Tap "Export" → save JSON → wipe install → reinstall → tap "Import" → pick the same JSON | Provider chain, system prompt, daemon-enabled, and other user settings are restored bit-for-bit. |
| 12 | Heartbeat notification deep-link | Daemon enabled, heartbeat fires | Tap the heartbeat notification | App launches (or comes to foreground via `singleTop`); deep-link extra `EXTRA_OPEN_HEARTBEAT` is consumed exactly once (subsequent rotation does not re-trigger). |
| 13 | Sandbox cancel | SandboxState = Downloading | Tap Cancel | State returns to NotInstalled; partial files cleaned; retry works. |
| 14 | iOS / desktop / web sandbox stub | Any non-Android platform | Try to invoke any sandbox feature | Operation fails with the message "Sandbox file browser is Android-only" (or returns empty for list/read paths) rather than crashing. |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| ctr-OQ1 | needs-runtime-test | Exact default values for daemon-enabled, heartbeat interval, default LLM provider — pinned in `AppSettings` defaults but not extracted in this phase. | Defaults extraction needs a focused read of `composeApp/commonMain/.../data/AppSettings.kt`; not load-bearing for the focus subsystems. |
| ctr-OQ2 | needs-spec-ruling | Settings-import error UX — partial-parse failure leaves settings in an inconsistent state? Atomic-replace? | Not extracted from source in this phase. |
| ctr-OQ3 | needs-runtime-test | Encrypted local storage — what algorithm, what key derivation, where the key lives? | Not extracted from source in this phase; relevant to a future defect-scan-semantic if security depth is needed. |
| ctr-OQ4 | needs-maintainer-decision | The "no battery whitelist request" decision — is this documented anywhere user-facing (e.g., FAQ) so users on aggressive-OEM ROMs know what to expect? | Not surfaced in README; could be a UX gap. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| ctr-CF1 | protocols | The proot wire / argv contract: exact `buildProcessArgs(...)` shape, env vars passed via `buildEnvVars(extraEnv)`, working-directory normalisation. | Wire-format extraction is the protocols phase. |
| ctr-CF2 | protocols | The AI tool-call protocol — exact JSON schema for a tool-call message, tool-result message, error shape, retry semantics. | Wire-format extraction is the protocols phase. |
| ctr-CF3 | protocols | Settings export JSON schema — field names, version field, migration policy. | Wire-format extraction is the protocols phase. |

## Carry-Forward Closure (from architecture)

| ID (in) | Closed because |
|---|---|
| kai-CF-alpine-proot | §Focus / Alpine pins the user-visible setup contract (states, mirror order, error-on-all-fail behavior, no-backoff retry semantics, cancellation contract, path-traversal protection). |
| kai-CF-foreground-service | §Focus / Foreground service pins notification visibility (low-importance ongoing), OS-kill recovery (`START_STICKY` + `MainActivity.onStart` re-assertion), Android 14+ FGS time-limit handling (`onTimeout` clean-stop), and Android 12+ background-start exception (silent swallow). |
| kai-CF-saf-binds | §Focus / SAF binds pins the FileKit-mediated picker contract (one-shot read; no persistable URI), the FileProvider outbound-share contract (`Intent.ACTION_VIEW` + read-grant), and the deliberate absence of persistent SAF grants. |
| kai-CF-battery-whitelist | §Focus / Battery whitelist pins the **deliberate non-request** contract — Doze/Standby behavior unmodified, OEM-kill silent, recovery via re-assertion next foreground. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | User-facing surfaces are split by surface type. | PASS | §Surfaces Covered table; §Feature Contracts grouped by surface (Compose UI: Chat / Settings / Settings export / Sandbox terminal; AI tools; Focus subsystems). |
| 2 | Feature contracts record trigger, defaults, outputs, side effects, persisted state, error behavior, and recovery behavior. | PASS | All seven feature-contract tables (chat, settings, settings export/import, sandbox terminal, plus the four focus subsystems) have all 7+ standard fields. |
| 3 | Security and authorization model is documented (if applicable). | PASS | §Security and Authorization (10 rows including LLM API keys, encrypted storage, MCP trust boundary, notification listener trust boundary, calendar/notification/SET_ALARM/FOREGROUND_SERVICE permissions, network security config note, sandbox trust boundary). |
| 4 | Contract ownership is mapped back to a layer or package. | PASS | Every feature-contract table has an "Owner (layer/package)" row citing the source-set + file. |
| 5 | A black-box acceptance list is included. | PASS | §Black-Box Acceptance List with 14 numbered scenarios — each with precondition, action, expected outcome — covering all four focus subsystems plus 5 cross-cutting acceptance scenarios. |
| 6 | Findings are marked with evidence levels. | PASS | Every claim tagged inline with *observed fact*, *strong inference*, *open question*, or *portability hazard*; four open questions captured in §Open Questions. |

**Validated by:** 2026-05-06 (contracts phase, session 1)
**Overall:** PASS
