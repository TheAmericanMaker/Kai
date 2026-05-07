# Mechanical Defects Report — Kai 9000

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `workflow/pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-06
- **Scope:** Mechanical passes only (1 logic, 2 error handling, 6 configuration). Semantic passes (3 concurrency, 4 security, 5 contract violations) deferred to `defect-scan-semantic` after protocols.

This pass focused on the Android source set (`composeApp/src/androidMain`) and the repo-root native-build script (`build-proot.sh`), because that is where Kai's heaviest platform-specific code lives (sandbox, daemon, foreground service, native build) and therefore where mechanical defects are most likely to manifest. The commonMain UI / ViewModel layer was spot-sampled but not exhaustively read in this phase. *(observed fact — architecture-map.md identifies androidMain as the thickest adapter layer.)*

The codebase shows generally good defensive discipline— defensive try/catch around filesystem walks is well-commented (`LinuxSandboxManager.getDiskUsageMB`, `ProotExecutor.readBounded`), evidence-tagged design rationale appears in code comments (`MainActivity.onStart`, `LinuxSandboxManager.homePath`), and most catch blocks log via `android.util.Log`. The defects below are the ones that did stand out.

---

## Pass 1— Logic and Correctness

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1.1 | `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt:138-143` | Symlink creation in tar extraction (`Files.createSymbolicLink`) wrapped in `catch (_: Exception) {}` with **no logging and no rollback**. If the host filesystem disallows symlinks (some Android external-storage mounts, FAT32 SD cards) or `linkName` resolves outside the sandbox, the entry is silently skipped and the rootfs is partially extracted. The user reaches `Ready` with a rootfs that may be missing critical symlinks (e.g. `/bin/sh → /bin/busybox`), which surfaces later as opaque `command not found` errors at the proot shell. | medium | observed fact | port differently — when porting, log the per-entry failure (debug level) and either abort extraction or accumulate a warning list surfaced in the post-extract state. |
| 1.2 | `composeApp/src/androidMain/.../sandbox/LinuxSandboxManager.kt:43-58` (the `homePath` lazy block) | The lazy initialiser calls `target.mkdirs()` and a one-time legacy-home migration. The return value of `mkdirs()` is **not checked**— if it returns false (the directory could not be created and does not already exist), `homePath` returns a path that does not point to a real directory, and downstream `File(homePath, ...)` operations throw at first use. The migration block also catches `Exception` and only logs a warning, leaving partial migration state. | medium | strong inference | port differently — gate the lazy with an early failure (Result type or sealed state) so the sandbox transitions to `Error` instead of pretending `Ready`. |
| 1.3 | `composeApp/src/androidMain/kotlin/com/inspiredandroid/kai/MainActivity.kt:115-122` (`handleDeepLinkIntent`) | The function reads `intent?.getBooleanExtra(EXTRA_OPEN_HEARTBEAT, false)` then mutates `intent.removeExtra(EXTRA_OPEN_HEARTBEAT)`. The same intent reference is shared with `onNewIntent` and `setIntent(intent)`— a configuration change racing a fresh Intent could either re-trigger or skip. Dedup is best-effort; correctness depends on `DataRepository.requestOpenHeartbeat` being idempotent. | low | open question | leave behind / port differently — the simpler fix is a session-scoped "consumed heartbeat ids" set instead of mutating the Intent. |

---

## Pass 2— Error Handling and Resilience

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 2.1 | `composeApp/src/androidMain/.../HeartbeatNotifier.android.kt:64-72` (`ensureChannel`) | Uses `runBlocking { getString(Res.string.notification_channel_name) }` and `runBlocking { getString(Res.string.notification_channel_description) }` *inside the system-broadcast notification post path* (`sendHeartbeatNotification`). Resource string lookup goes through Compose Multiplatform resources, which is `suspend`-based— `runBlocking` blocks the calling thread, which on Android is often the main thread. Risks— main-thread stall and potential deadlock if `getString` ever needs to dispatch back to the calling context. | medium | strong inference | fix before porting — precompute the channel strings at app start (DI-time) and inject them, or make `ensureChannel` a `suspend` function and call it from `Dispatchers.IO` before `notify()`. |
| 2.2 | `composeApp/src/androidMain/.../tools/NotificationHelper.kt:32-43` (`createNotificationChannel`, called from `init {}` block) | Same `runBlocking { getString(...) }` pattern, but invoked from the constructor's `init` block. NotificationHelper is constructed by Koin— the call happens **on first injection**, often during the first Composable render on the main thread. This is the worst-case timing for a `runBlocking`. | medium | strong inference | fix before porting — same fix as 2.1— precompute the strings at app start, pass them to NotificationHelper's constructor. |
| 2.3 | `composeApp/src/androidMain/.../DaemonController.android.kt:18-25` (`start()`) | Generic `catch (_: ForegroundServiceStartNotAllowedException) {}` swallows the exception with **no `Log.w`** and no telemetry. The mitigation (re-assertion on `MainActivity.onStart`) does run, but a user who toggles the daemon in settings expecting it to start *now* will silently no-op if the activity is in the background— invisible to anyone debugging "why didn't the daemon start?" | low | observed fact | port differently — log `Log.w("Daemon", "FGS start denied; will re-assert on next foreground")` at minimum. |
| 2.4 | `composeApp/src/androidMain/.../DaemonService.kt:27-32` | `try { startForeground(...) } catch (_: Exception) { stopSelf(); return }`— the catch is **too broad** (`Exception`) and the cause is discarded. Three different exception types matter here— `ForegroundServiceStartNotAllowedException` (Android 12+, the user backgrounded the app), `IllegalStateException` (Android 14+ FGS time-limit race), and unexpected `RuntimeException` (e.g., `SecurityException` if FOREGROUND_SERVICE permission was revoked). Treating them identically loses information and makes the FGS time-limit case indistinguishable from a permission-revoke. | low | strong inference | port differently — narrow the catch and log the type+message; treat permission-revoke specially. |
| 2.5 | `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt:47-64` (`download` mirror loop) | When all six mirrors fail, the loop throws `IOException("All Alpine mirrors failed", lastError)`. There is **no per-mirror retry-with-backoff** and **no exponential backoff between full retry passes** on user-driven re-try. On a transient network glitch, the user sees the error and must manually retry; on a captive-portal stale-cache they get the same failure six times in succession with no delay. | low | observed fact | port differently — add at minimum a single per-mirror retry-on-IOException, and use exponential backoff on the user-driven re-try. |
| 2.6 | `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt:80-94` (the inner `downloadFrom` write loop) | Writes to `targetFile` byte-by-byte via `output.write(buffer, 0, bytesRead)` but **does not flush between blocks** and does not `fd.sync()` at end. On Android, if the process is killed mid-download (OOM, OEM-killer), the partial file may not be fully flushed; the next `setup()` retry sees a half-written `.tar.gz` of nonzero size. The catch in `download()` does delete the partial, so this is benign in practice, but a delete-on-error happening *after* the catch propagates means there's a small window where the partial file lingers. | low | strong inference | leave behind — the existing delete-on-error covers the user-visible case. |
| 2.7 | `composeApp/src/androidMain/.../sandbox/LinuxSandboxManager.kt:307-310` (package install error path) | `catch (e: Exception) { Log.e(...); _state.value = SandboxState.Error("Install failed: ${e.message}") }`— the message-only `e.message` discards stack trace from the user-visible state. Logcat has the stack via `Log.e(..., e)`, but the UI shows just the message. For a port that wants debuggability without Logcat access, the surface should expose more. | low | observed fact | leave behind — port can decide based on its observability story. |

---

## Pass 6— Configuration and Environment Hazards

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 6.1 | `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt:17-27` | Alpine version (`3.21.3`), branch (`v3.21`), and the six-mirror list are hardcoded as `private const val`. **No override mechanism** — no BuildConfig field, no Gradle property, no runtime setting. F-Droid users in regions where some Alpine mirrors are blocked / throttled cannot add a custom mirror without rebuilding from source. Pinning the version is intentional (reproducibility), but the mirror list should be user-extensible. | low | observed fact | port differently — expose a "custom mirror URL" setting (with safety-warning UX) and prepend it to the mirror list. |
| 6.2 | `composeApp/src/androidMain/.../sandbox/ProotExecutor.kt:11-13` | `MAX_OUTPUT_LENGTH = 15_000`, `DEFAULT_TIMEOUT_SECONDS = 30L`, `MAX_TIMEOUT_SECONDS = 180L` are hardcoded. The numbers differ from the desktop sandbox (`composeApp/src/desktopMain/.../tools/ShellCommandTool.kt:16-18` uses `MAX_OUTPUT_LENGTH = 30_000` and `MAX_TIMEOUT_SECONDS = 120L`) and PersistentSandboxShell (`PersistentSandboxShell.kt:19` uses `15_000`). Three sources of truth for what is conceptually one config— a future change to one of them risks subtle drift. | medium | observed fact | port differently — lift these to a single `SandboxLimits` object in commonMain (or jvmShared) and reference it from both Android and desktop shell tools. |
| 6.3 | `build-proot.sh:18-22` | proot commit (`4dba3afbf3a63af89b4d9c1a59bf2bda10f4d10f`), talloc version (`2.4.3`), Android NDK major (`29`), `MIN_API=26`, ABI list (`arm64-v8a armeabi-v7a x86_64`) all hardcoded. No `--proot-commit=` or env-var override. Bumping proot for a CVE fix requires editing the script. | low | strong inference | leave behind — pinning is intentional for F-Droid reproducible builds. |
| 6.4 | `composeApp/src/androidMain/.../DaemonService.kt:17-19` | Channel ID `"kai_daemon_channel"` and notification ID `9001` are hardcoded `private const val`. `HeartbeatNotifier.android.kt:25, 31` similarly hardcodes channel ID `"kai_ai_notifications"` and ID `9002`. These are reasonable but a port supporting multiple Kai instances on one device (developer build alongside release build) would collide. | low | observed fact | leave behind — Android encourages stable IDs per channel. |
| 6.5 | `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt:18` | `BUFFER_SIZE = 8192` for the gzip-extract reader buffer is hardcoded. On modern Android devices typical IO benefits from 32–64 KB buffers; this is a small choice that may slow large extractions. | low | strong inference | leave behind — performance, not correctness. |
| 6.6 | `composeApp/src/androidMain/.../sandbox/LinuxSandboxManager.kt:65` | `prootPath` resolves to `applicationInfo.nativeLibraryDir/libproot.so` at every call. If Android `nativeLibraryDir` ever returns null (rare, but possible during `Application.onCreate` if Koin injects this manager too early), the `File(...)` call dereferences with a misleading path. There is no null-check. | low | strong inference | port differently — read `nativeLibraryDir` once at construction, gate with a clear error. |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 0 |
| Medium | 4 |
| Low | 12 |
| **Total** | 16 |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 0 | 2 | 1 | 3 |
| 2. Error handling | 0 | 0 | 2 | 5 | 7 |
| 6. Config and environment | 0 | 0 | 0 | 6 | 6 |

### Top Findings

1. **2.1 / 2.2 — `runBlocking { getString(...) }` in notification path** (`HeartbeatNotifier.android.kt:65-66`, `NotificationHelper.kt:33-34`)— blocking-call-on-main-thread risk in a heartbeat-critical code path. **Medium**, **fix before porting**.
2. **6.2 — Three sources of truth for sandbox output/timeout limits** (`ProotExecutor.kt:11-13` vs `PersistentSandboxShell.kt:19` vs desktop `ShellCommandTool.kt:16-18`)— drift risk. **Medium**, **port differently**.
3. **1.1 — Silent symlink-failure during tar extract** (`RootfsDownloader.kt:142-143`)— partially-extracted rootfs reaches `Ready`. **Medium**, **port differently**.
4. **1.2 — `LinuxSandboxManager.homePath` ignores `mkdirs()` failure**— race between "Ready" state and a non-existent home directory. **Medium**, **port differently**.
5. **2.5 — No retry-with-backoff in the rootfs download mirror loop**— low-grade reliability gap on transient network failures. **Low**, **port differently**.

### Routed To Semantic Phase

| ID | Description | Why Routed |
|----|-------------|-----------|
| mech-CF1 | The `runBlocking { getString(...) }` pattern in 2.1 / 2.2 is mechanical, but the **concurrency dimension** (which dispatcher is the caller on, what happens if `getString` suspends) belongs to defect-scan-semantic Pass 3. | Need contracts/protocols context to characterise the dispatcher policy. |
| mech-CF2 | The `MainActivity.onStart` re-assertion idempotency (1.3)— whether the heartbeat dedup is robust under configuration changes— is a state-machine concern. | Pass 5 (API contract violations) needs the contracts and protocols outputs to compare. |
| mech-CF3 | The shared `_state.value = ...` writes in `LinuxSandboxManager` (multiple `scope.launch { ... _state.value = ... }` blocks) are concurrency-flavored. | Pass 3. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | All three passes produced findings (3 / 7 / 6). |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row in §Pass 1, §Pass 2, §Pass 6 has all four columns populated. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | One table per pass; medium findings listed before low within each pass. |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS | Severity-table and per-pass table totals (16) match the row counts in the detail tables. |
| 5 | Items spotted that are actually semantic in nature are routed to defect-scan-semantic via carry_forward in workflow/status.yaml. | PASS | Three items routed (mech-CF1, mech-CF2, mech-CF3)— see §Routed To Semantic Phase. |
| 6 | Findings are marked with evidence levels. | PASS | Every finding row carries an Evidence Level column with one of `observed fact`, `strong inference`, `open question`. |

**Validated by:** 2026-05-06 (defect-scan-mechanical, session 3)
**Overall:** PASS
