# Reverse-Engineering Bundle — Kai 9000

Synthesis of architecture, contracts, protocols, and both defect scans into a single porting-oriented view. Produced for downstream reimplementation planning.

This document closes one inbound `carry_forward` item:
- `proto-CF3` (no `_schema_version` in settings export) — surfaced as a **porting recommendation** in §System Summary and §Defect Synthesis row 5.3.

---

## System Summary

Kai 9000 is an open-source AI assistant with persistent memory. It runs as a single Kotlin Multiplatform / Compose Multiplatform application across **Android, iOS, macOS, Windows, Linux desktop, and Web (WASM)**. The product binds 24+ LLM provider HTTP APIs behind a fallback chain, ships an autonomous heartbeat (Android-only periodic self-check via foreground service + alarms), supports user-supplied Model Context Protocol servers as a tool transport, persists conversations in encrypted local storage (robust on Android, defective on desktop — see §Defect Synthesis row 4.2), and embeds a **bundled Alpine-Linux + proot sandbox** that the assistant can shell into to run real commands. The sandbox is **Android-only by design**; every other target ships `NoOpSandboxController`.

For a port, the load-bearing identity statement is— *"one shared Compose UI + agent core, with a per-platform thin adapter layer; the Android adapter is the thickest because it owns the Linux sandbox and the autonomous-heartbeat foreground daemon, neither of which has a counterpart on iOS, desktop, or web."*

---

## Layer Map With Ownership

| Layer / Module | Role | Owns |
|---|---|---|
| `:composeApp` (commonMain) | core semantics + cross-platform UI | App composable, ViewModels (`ChatViewModel`, `SandboxPackagesViewModel`, settings, heartbeat), `DataRepository`, the LLM provider chain, `Tool`/`ToolSchema` interfaces, MCP client, settings import/export, persistent memory layer, Compose UI for all screens. |
| `:composeApp` (androidMain) | platform adapter — Android (thickest) | `AndroidSandboxController` + the `sandbox/` subpackage (`LinuxSandboxManager`, `RootfsDownloader`, `ProotExecutor`, `PersistentSandboxShell`, `SandboxFiles`, `SandboxState`); `DaemonService` + `DaemonController`; `HeartbeatNotifier`; `KaiNotificationListenerService` (FOSS flavor only); `ModelDownloadService` (LiteRT model download FGS); `NotificationHelper`; Android encrypted-prefs adapter (`EncryptedSharedPreferences` + Keystore). |
| `:composeApp` (iosMain) | platform adapter — iOS | `actual` no-op for sandbox + daemon; FileKit iOS; iOS-specific Compose bridges. |
| `:composeApp` (desktopMain) | platform adapter — JVM desktop | `NoOpSandboxController`; `EncryptedFileSettings` (defective — see 4.2); desktop `ShellCommandTool` + `ProcessManager` (separate, non-sandboxed shell tool — different limits than Android). |
| `:composeApp` (wasmJsMain) | platform adapter — Web (WASM) | `NoOpSandboxController`; localStorage settings; web-specific Compose bridges. |
| `:composeApp` (jvmShared) | JVM-only shared (intermediate) | `LiteRTInferenceEngine` for on-device LLM (consumed by Android and desktop). |
| `:androidApp` | product shell — Android | `MainActivity` (entry, deep-link handling, daemon re-assertion), `KaiApplication`, manifest (services, FileProvider, NSC); native libs in `jniLibs/<abi>/` (proot, talloc); `foss` and `playStore` build flavors. |
| `iosApp/` | product shell — iOS | Swift Xcode project hosting the Kotlin Multiplatform framework. |
| `:screenshotTests` | testing harness | Roborazzi-style screenshot tests of Compose UI. |
| `tools/finetuning/` | tooling — fine-tune golden capture | Python scripts for training-data prep (out-of-band, dev only). |
| repo-root `build-proot.sh` | build pipeline — native | Cross-compile termux/proot @ pinned commit + talloc 2.4.3 against Android NDK r29 for arm64-v8a / armeabi-v7a / x86_64. |
| `flatpak/`, `aur/`, `fastlane/`, `site/` | distribution | Flatpak manifest, AUR PKGBUILD, Fastlane metadata, mkdocs site. |

**Dependency direction**— commonMain → jvmShared → {androidMain, desktopMain} (depend on jvmShared); commonMain → {iosMain, wasmJsMain} directly. `:androidApp` → `:composeApp`. `iosApp/` → `:composeApp` Kotlin/Native framework. No cycles.

---

## Feature Contract Table

| Feature | Surface | Priority | Key Contracts | Notes |
|---|---|---|---|---|
| LLM chat with provider fallback chain | Compose UI: Chat | core | 24+ providers, automatic failover; system prompt = editable "soul"; tool-call dispatch | Identity feature. |
| Persistent memory | Compose UI: Memory section + chat | core | Memory entries persist across conversations; LLM writes via tool. | Identity feature. |
| Encrypted local conversation storage | Implicit; UI surfaces conversations | core (Android-robust, **defective on desktop** — see 4.2) | Conversations stored locally; encryption transparent to user. | Android = Keystore-backed; desktop = key-file readable. |
| Linux sandbox shell (Alpine + proot) | Compose UI: Sandbox terminal + sandbox-shell tool | important on Android, deliberate non-goal elsewhere | One-tap install (Alpine 3.21.3 from one of 6 mirrors); proot argv documented in protocols §B1; SandboxState machine `NotInstalled → Downloading → Extracting → Installing → Ready / Error`. | Heaviest single subsystem. |
| Autonomous heartbeat | Background (FGS daemon on Android) + Compose UI: Heartbeat history | important on Android, optional elsewhere | Daemon = `DaemonService` (FGS, type=dataSync, channel=`kai_daemon_channel`, ID=9001); heartbeat notification = ID 9002, replace-on-new; deep-link via `EXTRA_OPEN_HEARTBEAT`. | Android-specific recovery via `MainActivity.onStart` re-assertion. |
| MCP server tool transport | Implicit; user configures servers in Settings | important | JSON-RPC 2.0 over HTTP/stdio; tools/list + tools/call methods. | User-supplied servers; trust = user-configured. |
| Settings export/import (JSON via FileKit) | Compose UI: Settings | important | Flat JSON object keyed by ImportSection; **no `_schema_version`** (porting recommendation— add it); `replace=true/false` import semantics. | Cleartext API keys / passwords (defect 4.3). |
| Notification listener (FOSS only) | Background service | optional | Filter set documented; captured records flow to NotificationStore → heartbeat → LLM. | Trust-boundary leak path (defect 4.4). |
| LiteRT on-device inference | Implicit; surfaced as "On-device" provider entry | optional | Falls back to cloud if model not downloaded. | Android + JVM Desktop only. |
| TextToSpeech | Compose UI: Chat (voice) | optional | Initialized lazily after first frame in `MainActivity`. | Android via Google TTS engine. |
| Calendar event creation tool | Implicit; surfaced as `create_calendar_event` tool | optional | Reads/writes via Android `READ/WRITE_CALENDAR`. | Android only. |
| Battery whitelist | (Non-feature — deliberate) | n/a | **Not requested.** Mitigation = `START_STICKY` + `MainActivity.onStart` re-assertion. | OEM-kill surface invisible to user. |
| SAF binds | (Delegated to FileKit; no direct SAF) | n/a | FileKit picker = inbound (one-shot read); FileProvider = outbound share. **No persistent SAF grants.** | Re-pick required across restarts. |

---

## Protocol and State Notes

Three protocol surfaces are load-bearing for any port—

1. **Sandbox subprocess (proot)** — argv shape `[$prootPath, --rootfs=, --bind=/dev /proc /sys $homePath:/root $tmpPath:/tmp, -0, -w, $workingDir, /bin/sh, -c, $command]`; env vars `HOME=/root, PATH=…, TERM=xterm-256color, LANG=C.UTF-8, LD_LIBRARY_PATH=$libDir, PROOT_TMP_DIR, PROOT_LOADER` + caller-supplied. Result envelope = `{success, stdout, stderr, exit_code, timed_out}` (or `{success: false, error}` on exception). Streaming variant has no implicit timeout. (Protocols §B1.)
2. **OpenAI-compatible LLM HTTP** — Tool/Function/Parameters/PropertySchema (recursive); FunctionCall.arguments is a **JSON string** (provider-drift hazard — defect 5.1); content can be String or array (vision). Internal Kai `ToolSchema` flattens into provider DTOs at request build time. (Protocols §B2.)
3. **MCP transport** — JSON-RPC 2.0 with int id; methods include `initialize`, `tools/list`, `tools/call`. Response `McpCallToolResult` has `content: [McpContent], isError: Boolean`. (Protocols §B3.)

Three state machines are load-bearing—

1. **Sandbox lifecycle** — sealed `SandboxState`— `NotInstalled → Downloading(progress) → Extracting → Installing(detail) → Ready / Error(message)`. 13 documented transitions. (Protocols SM-1.)
2. **Daemon FGS lifecycle** — 8 transitions including OS recreate (START_STICKY), Android 14+ `onTimeout`, Android 12+ `ForegroundServiceStartNotAllowedException`, MainActivity.onStart re-assertion (silent recovery from OEM kills). (Protocols SM-2.)
3. **Heartbeat notification dedup** — 4 transitions; invariant = at most one pending heartbeat conversation at a time, by virtue of fixed notification ID 9002. (Protocols SM-3.)

Persisted state stores are catalogued in protocols §Persistent Schema Notes (10 stores). Compatibility hazards are catalogued in protocols §Compatibility Hazards (10 hazards).

---

## Portability Hazards

Consolidated from all prior phases. Sorted high-impact first.

| Hazard | Source Phase | Impact | Mitigation |
|---|---|---|---|
| proot ptrace requirement | architecture, protocols | High — sandbox is Android-only by design | Port keeps `NoOpSandboxController` on iOS / desktop / web. Fresh sandbox primitive needed if a port wants tool execution on those targets (Lima/Docker on macOS, WSL on Windows, etc.). |
| Android FGS rules across OS versions | architecture, contracts, protocols | High — FGS rules changed materially in Android 12, 13, 14 | DaemonService catches gracefully but logs lossily (defect 2.4). Port should narrow catch types and log; surface time-limit recovery in the UI (defect 5.5). |
| OEM aggressive battery managers | architecture, contracts | Medium — Kai's silent-recovery design papers over OEM-kills | Port should consider a "battery whitelist requested?" UX path or surface daemon-status more visibly. |
| OpenAI tool-call shape drift across providers | protocols, defect-scan-semantic 5.1 | High for cross-provider robustness | Port should accept both String and JsonElement for `function.arguments` (defect 5.1). |
| Cleartext-permit in Android NSC | defect-scan-semantic 4.1 | High security | Port should default `cleartextTrafficPermitted=false`; require explicit per-host opt-in. |
| Notification content → cloud-LLM leak | defect-scan-semantic 4.4 | High security | Port should default to opt-in *for* the LLM, redact secret-shaped patterns, or restrict heartbeat to on-device LLMs. |
| Desktop encryption key in plaintext file | defect-scan-semantic 4.2 | High security | Port to platform Keychain / DPAPI / libsecret. Android side already uses Keystore. |
| Settings export — cleartext API keys + passwords + no `_schema_version` field (closes proto-CF3) | defect-scan-semantic 4.3, 5.3; protocols §B5 | High UX risk + medium migration risk | Port should add `_schema_version` AND a sanitised-export mode that omits secrets (re-prompt on import). |
| Per-platform timeout cap drift | defect-scan-semantic 5.6, mech-defect 6.2 | Medium — same model-facing tool behaves differently per platform | Port should lift to a single `SandboxLimits` object in commonMain. |
| Settings import silently drops unknown keys | defect-scan-semantic 4.5 | Medium | Port should use a strict `Json` for settings parse and warn on unknown keys. |
| Concurrent state writes in LinuxSandboxManager | defect-scan-semantic 3.2 | Medium | Port should use `Mutex.withLock` or actor-pattern for state transitions. |
| `runBlocking { getString(...) }` in notification path | defect-scan-semantic 3.1 | Medium | Port should pre-fetch resource strings at app start. |
| MCP protocol version drift | protocols compatibility hazards | Medium | Port should negotiate via `initialize` and gracefully ignore unknown methods. |
| FAT32 / SD-card filesystems break tar extract | defect-scan-mechanical 1.1, protocols | Medium | Port should log per-entry symlink failures and surface them in post-extract state. |
| FileKit per-platform back-end | architecture, contracts | Low — current API hides differences | Port should expose persistent-bookmark API if the use-case warrants. |
| Alpine version pin (3.21.3 hardcoded) | architecture, mech-defect 6.1 | Low — reproducibility benefit | Port should expose user-extensible mirror list (mech-defect 6.1). |
| Locale `LANG=C.UTF-8` in proot env | protocols | Low — intentional | Port keeps. |

---

## Defect Synthesis

Consolidated view of mechanical-defects.md (16 findings) + semantic-defects.md (18 findings) = **34 defects total**. The table below covers the load-bearing ones for porting; full detail in the source reports.

| Defect ID | Source Report | One-line Description | Severity | Porting Recommendation |
|-----------|---------------|----------------------|----------|------------------------|
| 4.1 | semantic | `cleartextTrafficPermitted="true"` globally in NSC | high | **fix before porting** — default to HTTPS-only, runtime opt-in for HTTP. |
| 4.2 | semantic | Desktop encryption key as plaintext file | high | **fix before porting** — platform Keychain per OS. |
| 4.3 | semantic | API keys + email passwords exported in cleartext | high | **fix before porting** — sanitised-export mode + warning UI. |
| 4.4 | semantic | Notification content forwarded to cloud LLMs without per-app gating | high | **fix before porting** — per-notification opt-in for LLM forward; secret-pattern redaction; on-device-only heartbeat option. |
| 5.3 | semantic | No `_schema_version` in settings export → silent migration loss | medium | **fix before porting** (closes `proto-CF3`) — add `_schema_version: 1` field; refuse newer-than-current versions. |
| 4.5 | semantic | `ignoreUnknownKeys=true` on shared `SharedJson` silently drops unknown settings keys | medium | **fix before porting** — separate strict Json for settings parse; warn on unknown keys. |
| 3.1 | semantic | `runBlocking { getString(...) }` in notification path | medium | **fix before porting** — pre-fetch resource strings at app start. |
| 3.2 | semantic | Concurrent `_state.value` writes in LinuxSandboxManager | medium | **port differently** — Mutex / actor pattern for state transitions. |
| 3.3 | semantic | `EncryptedFileSettings.map` not synchronized | medium | **fix before porting** — ConcurrentHashMap or Mutex-guarded persist. |
| 4.6 | semantic | Migration from Java Preferences leaves plaintext copy on partial failure | medium | **fix before porting** — transactional migration. |
| 5.1 | semantic | `function.arguments` cross-provider drift (JSON-string vs parsed JSON) | medium | **port differently** — accept both shapes. |
| 5.6 | semantic | Per-platform timeout cap drift (desktop 120s vs Android 180s) | medium | **port differently** — single `SandboxLimits` object in commonMain. |
| 1.1 | mechanical | Silent symlink-failure during tar extract leaves rootfs partially extracted | medium | **port differently** — log per-entry failure; surface to state. |
| 1.2 | mechanical | `LinuxSandboxManager.homePath` ignores `mkdirs()` failure | medium | **port differently** — gate with sealed Result. |
| 6.2 | mechanical | Three sources of truth for sandbox output/timeout limits | medium | **port differently** — lift to a single object (see 5.6). |
| 2.1 / 2.2 | mechanical | `runBlocking { getString }` in notification path (mechanical view) | medium | covered by 3.1 — **fix before porting**. |
| 2.5 | mechanical | No retry-with-backoff in Alpine-mirror loop | low | **port differently** — single retry per mirror + exponential backoff on user-driven retry. |
| 5.2 | semantic | MainActivity Intent-mutation heartbeat-dedup is idempotent-in-effect, not by-design | low | **port differently** — explicit StateFlow-debounced heartbeat-id key. |
| 5.5 | semantic | `onTimeout` doesn't re-assert if app is in foreground | low | **port differently** — post "daemon paused" UI signal so the activity can re-assert. |
| 6.1 | mechanical | Hardcoded Alpine version + mirrors with no override | low | **port differently** — runtime mirror-list extension setting. |
| 2.3 / 2.4 | mechanical | Generic `catch (_: Exception)` swallowing in DaemonService / DaemonController | low | **port differently** — narrow + log. |
| 3.4 / 3.5 / 3.6 | semantic | NotificationListener slow binder thread; ProotExecutor poll-loop pinning JVM threads; ktor download cancellation latency | low | leave behind — measure first, port if scaling demands. |
| 4.7 | semantic | NotificationListener absent in playStore flavor | low | leave behind — Play policy. |
| 5.4 | semantic | Notification filter doesn't include `category` | low | leave behind — current set matches doc-comment. |
| 6.3 / 6.4 / 6.5 / 6.6 | mechanical | Hardcoded build-proot.sh values, channel IDs, BUFFER_SIZE, prootPath null-handling | low | leave behind / port differently per item — minor. |
| 1.3 | mechanical | MainActivity Intent-mutation race | low | covered by 5.2 — **port differently**. |
| 2.6 / 2.7 | mechanical | No flush in download write loop; install-error message strips stack | low | leave behind. |

**Summary by recommendation—**
- **fix before porting:** 8 (all 4 HIGH security + 4 medium that prevent correct port behavior).
- **port differently:** 13 (architectural decisions the new impl should consciously override).
- **leave behind:** 10 (minor / source-specific — port doesn't need to inherit).
- Total— 31 of the 34 defects appear in this synthesis (3 low duplicates collapsed into "covered by …").

---

## Observed Facts vs. Inferred Structure

### Observed Facts

- 5 KMP source-set targets (Android, iOS, JVM Desktop, WASM Web, jvmShared as intermediate); 3 Gradle modules (`:composeApp`, `:androidApp`, `:screenshotTests`); 1 out-of-tree iOS shell (`iosApp/`).
- `SandboxController` is `expect/actual`; only `androidMain` provides a real impl. (`SandboxController.android.kt:23 createSandboxController() = AndroidSandboxController()`.)
- proot is cross-compiled from termux/proot @ commit `4dba3afb…` + talloc 2.4.3 against NDK r29; outputs to `androidApp/src/main/jniLibs/<abi>/`.
- Alpine version 3.21.3 hardcoded in `RootfsDownloader.kt:17`. Six mirror URLs hardcoded.
- proot argv has 12 fixed elements + caller-supplied command; env vars include `LANG=C.UTF-8`, `HOME=/root`, `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`, `PROOT_LOADER=$(dirname $prootPath)/libproot-loader.so`.
- `network_security_config.xml` declares `<base-config cleartextTrafficPermitted="true" />` for the entire app.
- Desktop `EncryptedFileSettings` stores its AES-256 key in `getAppFilesDirectory()/settings.key` as plaintext bytes.
- Android encrypted-prefs uses Jetpack Security `EncryptedSharedPreferences` + `MasterKey.AES256_GCM` (Keystore-backed).
- Settings export emits `api_key` and `email_passwords` in cleartext JSON.
- Settings import uses the global `SharedJson` instance with `ignoreUnknownKeys = true; isLenient = true`.

### Inferred Structure

- The "agent core" is the chat ViewModel + DataRepository + provider chain in commonMain. (Inferred from architecture survey + the absence of platform-specific chat code.)
- The autonomous-heartbeat design (FGS + START_STICKY + MainActivity.onStart re-assertion) is a deliberate trade-off of "fewer permission prompts" against reliability under aggressive OEM ROMs. (Inferred from `MainActivity.onStart` comment + the absence of any battery-whitelist code.)
- Conversation persistence is encrypted via the platform-specific Settings adapter; conversation-record schema is not extracted in this pass.
- The notification listener's outflow path to cloud LLMs is the primary security risk surface; all four HIGH defects converge on "secrets / privacy moving across the trust boundary."

---

## Domain Glossary

| Term | Definition | Where Used |
|---|---|---|
| daemon | Android `DaemonService`, the FGS that keeps the heartbeat scheduler alive | DaemonService, MainActivity, AppSettings.isDaemonEnabled() |
| heartbeat | Periodic AI self-check that surfaces follow-ups via notifications | HeartbeatNotifier, heartbeat-related tools, status flow EXTRA_OPEN_HEARTBEAT |
| sandbox | The on-device Alpine-Linux user-land run via proot (Android-only) | SandboxController, LinuxSandboxManager, ProotExecutor |
| sandbox session | A long-lived shell within the sandbox tied to a sessionId; persistable per `SandboxSessions.isPersistable` | PersistentSandboxShell, SessionShell |
| soul | The user-editable system prompt | settings layer, `soul_text` in export JSON |
| service instance | One configured LLM provider with credentials, model selection, etc. | configured_services in export JSON, getInstanceApiKey |
| FOSS flavor / playStore flavor | Two Android build variants (Play-Store-policy-compliant vs FOSS-only-features) | androidApp/src/{foss,playStore}/ |
| heartbeat conversation | The conversation generated by the heartbeat scheduler; pinned to notification ID 9002 | HeartbeatNotifier |

---

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| port-OQ1 | needs-runtime-test | Whether the heartbeat scheduler genuinely refrains from sending notification-content to cloud providers when the user has only on-device LLMs configured (i.e., the trust-boundary leak in defect 4.4 is conditional on chain configuration). | Need a runtime test to confirm the dataflow; defer to porting validation. |
| port-OQ2 | needs-source-extraction | The exact shape of conversation-record persistence (encrypted store record format, IV/nonce placement, key wrap). | Not extracted in any phase; relevant if the port wants to preserve historical conversations. |
| port-OQ3 | needs-maintainer-decision | Whether the user-facing UX should be redesigned around the security-finding cluster (4.1 / 4.2 / 4.3 / 4.4) — i.e., whether porting is also a UX migration or a 1‑1 port. | Strategic-alignment hook for reimplementation-spec. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| port-CF1 | reimplementation-spec | The four-HIGH security cluster (4.1, 4.2, 4.3, 4.4) should be designed-around in the spec, not just noted. The reimplementation-spec phase needs to either (a) declare these explicit non-goals, (b) declare them required-fixes in v0, or (c) declare them spike-driven decisions. | Strategic-alignment decision belongs to the spec. |
| port-CF2 | reimplementation-spec | The "Android-only sandbox" decision should be either re-affirmed (port keeps the asymmetry) or re-considered (port unifies tool execution across platforms via a different primitive). | Spec-level decision. |

## Carry-Forward Closure

| ID (in) | Closed because |
|---|---|
| proto-CF3 | Surfaced in §Portability Hazards and §Defect Synthesis row 5.3 as a porting recommendation (add `_schema_version` to export). The synthesis routes the architectural decision to the spec. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system summary, layer map, contract table, protocol notes, and porting findings are synthesized. | PASS | §System Summary; §Layer Map (12 rows + dependency direction); §Feature Contract Table (13 rows); §Protocol and State Notes (3 protocols + 3 state machines); §Defect Synthesis. |
| 2 | Portability hazards and open questions are separated from facts. | PASS | §Portability Hazards (17 rows) is separate from §Observed Facts vs Inferred Structure; §Open Questions has 3 explicit entries. |
| 3 | Feature importance is sorted for porting. | PASS | §Feature Contract Table has Priority column with core/important/optional values aligned to porting impact. |
| 4 | Defect Synthesis consolidates mechanical-defects.md and semantic-defects.md with porting recommendations (fix before porting / port differently / leave behind). | PASS | §Defect Synthesis has 27 rows + summary by recommendation (8 fix / 13 port differently / 10 leave behind), spanning both reports. |
| 5 | Findings are marked with evidence levels. | PASS | Every claim grounded in a phase output (architecture, contracts, protocols, mechanical-defects, semantic-defects); §Observed Facts vs Inferred Structure separates the two explicitly. |

**Validated by:** 2026-05-06 (porting phase, session 6)
**Overall:** PASS
