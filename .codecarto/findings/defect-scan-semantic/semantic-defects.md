# Semantic Defects Report — Kai 9000

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Contracts reference:** `findings/contracts/behavioral-contracts.md`
- **Protocols reference:** `findings/protocols/protocols-and-state.md`
- **Mechanical defects reference:** `findings/defect-scan-mechanical/mechanical-defects.md`
- **Pipeline:** `workflow/pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-06
- **Scope:** Semantic passes only (3 concurrency, 4 security, 5 contract violations).

This phase inherits five `carry_forward` items targeting it— `mech-CF1` (runBlocking dispatcher hazard), `mech-CF2` (heartbeat-dedup contract robustness), `mech-CF3` (concurrent state writes in LinuxSandboxManager), `proto-CF1` (settings-import key-drop), `proto-CF2` (function.arguments cross-provider drift). Each is closed below.

---

## Pass 3— Concurrency and Resource Management

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 3.1 | `composeApp/src/androidMain/.../HeartbeatNotifier.android.kt:64-72` and `composeApp/src/androidMain/.../tools/NotificationHelper.kt:32-43` (closes `mech-CF1`) | `runBlocking { getString(Res.string.notification_channel_name) }` blocks the calling thread on a `suspend` resource lookup. NotificationHelper's `init {}` block runs on Koin first-injection — typically during the first Composable render on the **main thread**. `runBlocking` parks the main thread; if `compose.resources` `getString` ever needs to dispatch to `Dispatchers.Main.immediate` to load string resources, the body cannot make progress (the main thread is held). Likely lock-step OK in practice today (Compose Resources may resolve on the calling dispatcher), but it is a **potential deadlock** under any future Compose Resources change that introduces a Main hop. The HeartbeatNotifier path runs from the heartbeat scheduler (typically `Dispatchers.IO`), which is safer but still wastes IO threads. | medium | strong inference | fix before porting — pre-fetch the channel strings in commonMain at app start (Koin module init with `runBlocking` *there*, where it's unambiguously safe), inject the resolved strings into HeartbeatNotifier and NotificationHelper. Closes `mech-CF1`. |
| 3.2 | `composeApp/src/androidMain/.../sandbox/LinuxSandboxManager.kt` — multiple `scope.launch { ... _state.value = ... }` blocks throughout the file (closes `mech-CF3`) | `MutableStateFlow.value =` is documented as atomic per write; **read-modify-write** sequences (`if (state.value is Ready) { _state.value = NotInstalled }`) are not. The `setup()`, `cancel()`, `reset()`, and `installPackages()` paths can be invoked from different UI events; if two arrive nearly simultaneously (e.g., user taps "Cancel" while "Install Packages" is in flight), the state flow can land in an unintended state. There is **no `Mutex` or `actor` channel serialising state transitions**. | medium | strong inference | port differently — wrap state transitions in a `Mutex.withLock { ... }` or, more idiomatically in Kotlin, route mutations through a single-coroutine actor pattern (Channel-based command queue). Closes `mech-CF3`. |
| 3.3 | `composeApp/src/desktopMain/.../data/EncryptedFileSettings.kt`— the `map: MutableMap<String, String>` (line 25) is a plain `mutableMapOf()` accessed by every `Settings` get/set | Multiple threads can be in `get`/`set` concurrently (Settings are read in many places, written from settings UI events). The map has no synchronisation. Reads racing writes can throw `ConcurrentModificationException`; writes racing writes can lose updates. The full-file encrypt-and-write `persist()` is also not synchronised, so two concurrent writers can each compute different encrypted bytes and the last one wins. | medium | strong inference | fix before porting — use `ConcurrentHashMap` (or wrap with a `Mutex`) and serialise persist via a single-thread executor or a Mutex-guarded `withContext(Dispatchers.IO)` block. |
| 3.4 | `composeApp/src/androidMain/.../notifications/KaiNotificationListenerService.kt:onNotificationPosted` | The override runs on a system-bound binder thread. Inside, `lookupAppLabel(pkg)` is a synchronous `PackageManager` call (cold-cache slow). If the system posts notifications faster than the binder thread can drain them through `lookupAppLabel + record build`, Android may kill the listener for being slow. | low | strong inference | port differently — defer record building to the IO-scope coroutine; on the binder thread, capture only the cheap fields (`pkg`, `sbn.id`, `extras`) and post to a Channel. |
| 3.5 | `composeApp/src/androidMain/.../sandbox/ProotExecutor.kt:73-82` (`awaitExit`) | Uses a 200 ms `process.waitFor` poll loop to keep cancellation responsive. Each iteration spends 200 ms in a kernel timeout-wait; a long-running command holds the JVM thread for its full duration in 200 ms-sized blocks. On a heavily multitasked sandbox, dozens of `ProotHandle.awaitExit()` calls can pin a similar number of JVM threads. **Not a defect at low concurrency**; flagged as a scaling hazard. | low | open question | leave behind — measure first; only matters if the sandbox runs many concurrent shells. |
| 3.6 | `composeApp/src/androidMain/.../sandbox/RootfsDownloader.kt:82-94` (`downloadFrom` ktor read loop) | Uses `bodyAsChannel().readAvailable(buffer)` to drain a streamed download. Inside the inner loop there's no `coroutineContext.isActive` check; cancellation relies on Ktor honouring the parent scope cancellation, which it should — but cancellation latency is a function of how long `readAvailable` blocks on a single TCP read. On a slow remote, the user pressing Cancel may see a 30-second delay. | low | open question | leave behind. |

---

## Pass 4— Security and Trust Boundaries

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 4.1 | `androidApp/src/main/res/xml/network_security_config.xml` | The Network Security Config declares `<base-config cleartextTrafficPermitted="true" />`. **Cleartext (HTTP) traffic is permitted globally for the entire app, all hosts.** Despite the contracts-phase claim that "default is HTTPS-only", the actual config opts the app **out** of Android's HTTPS-default. Implications— (a) a user-configured LLM provider with `http://` works without warning; (b) an MCP server URL over plain HTTP is accepted — API keys / tokens travel in cleartext over public wifi; (c) the LLM provider chain has no preflight to flag HTTP endpoints. | high | observed fact | fix before porting — change the default to `cleartextTrafficPermitted="false"` and add per-host opt-ins (or remove the override entirely). For users who genuinely need HTTP (local Ollama at 127.0.0.1), expose a runtime opt-in setting that triggers a separate cleartext-permit per `domain-config`. |
| 4.2 | `composeApp/src/desktopMain/.../data/EncryptedFileSettings.kt:33-43` (`getOrCreateKey`) | The AES-256 key for desktop's encrypted settings is stored in `getAppFilesDirectory()/settings.key` as **plaintext bytes** with no OS-level access control. On Linux/macOS/Windows, `getAppFilesDirectory()` is per-user but readable by any process running as the same user — so any unprivileged malware on the same account can read `settings.key`, decrypt `settings.aes`, and exfiltrate every API key, email password, and credential. The "encrypted storage" feature, on desktop, only protects against another user on the same machine — not against malware. | high | observed fact | fix before porting — use the platform Keychain (macOS), DPAPI (Windows), libsecret/Secret Service (Linux). The Android side already does the right thing via Jetpack Security `EncryptedSharedPreferences` with `MasterKey.AES256_GCM` (hardware-bound via Keystore — `Platform.android.kt:160-168`). |
| 4.3 | `composeApp/src/commonMain/.../data/AppSettings.kt:765-787` (export of SERVICES section) | `exportToJson(...)` for the SERVICES section iterates each configured instance and emits `put("api_key", JsonPrimitive(getInstanceApiKey(instance.instanceId)))` — **API keys are exported in cleartext** in the JSON file the user picks. The "Settings export round-trip" contract (acceptance #11) says the chain is restored "bit-for-bit", so this is intentional, but the user is exporting their secrets to a file they may sync to cloud, email to support, or share with a friend. | high | observed fact | fix before porting — present an export-preview dialog that **explicitly warns** "this file contains your API keys in plaintext" and offer a "Sanitised export" mode that omits secrets (the imported sanitised file would re-prompt for keys). The same applies to `email_passwords` (`AppSettings.kt:831-836`). |
| 4.4 | `composeApp/src/androidMain/.../notifications/KaiNotificationListenerService.kt:onNotificationPosted` (per-notification capture) | Captured notification text is stored in `NotificationStore` and **fed to the heartbeat path**, which (per protocol B2) sends it to whichever LLM provider the user has configured. If the user has cloud providers in the chain, **notification content (potentially including 2FA codes, payment confirmations, message previews, security alerts) is sent off-device to a third-party provider**. The user's filter is at the OS level (Notification Access "Apps" picker); Kai does not separately ask "does the user want this app's notifications fed to the cloud LLM?" | high | strong inference | fix before porting — per-app gating at the Kai layer (defaults to opt-in *for* the LLM, not opt-out), redact obvious-secret-shaped patterns (6-digit number near "code" / "OTP"), or restrict heartbeat to on-device LLMs. |
| 4.5 | `composeApp/src/commonMain/.../network/Requests.kt:62-63` (HTTP client config) | `Json { isLenient = true; ignoreUnknownKeys = true }` is set globally on the network HTTP client. For LLM provider responses this is appropriate (providers add fields). However the **same `SharedJson` instance** is used by `RemoteDataRepository.importSettingsFromJson` (`RemoteDataRepository.kt:1949`), which calls `SharedJson.parseToJsonElement(json).jsonObject` — settings imports silently drop unknown keys. (Closes `proto-CF1`.) | medium | observed fact | fix before porting — use a separate, strict `Json` instance for settings parse (`ignoreUnknownKeys = false`); when it errors, surface a "this export contains unknown keys, some sections may be skipped" warning to the user. Closes `proto-CF1`. |
| 4.6 | `composeApp/src/desktopMain/.../data/EncryptedFileSettings.kt:69-83` (`migrateFromPreferences`) | Migration from Java `Preferences` does `persist()` *then* `prefs.clear()`. If the process crashes between the two — or if `prefs.clear()` throws and is caught silently (which the surrounding `catch (_: Exception) {}` does) — the **plaintext copy in `Preferences` survives** while the user thinks settings are now encrypted. Java Preferences on Linux are stored in `~/.java/.userPrefs` as XML; on macOS in `~/Library/Preferences`; on Windows in the registry. All are plaintext. | medium | strong inference | fix before porting — make migration transactional— write the new file, fsync, then clear preferences, then delete the old file. On any failure, the user is no worse off than pre-migration. Log the migration outcome. |
| 4.7 | Notification Listener service (FOSS flavor only — per the doc-comment "Registered only in the FOSS flavor manifest") | The class doc-comment says the listener is FOSS-only; if true, the **Play Store flavor** of Kai does not run a notification listener. The user installs Kai from Play Store thinking they have feature parity with the FOSS build but silently does not. | low | open question | leave behind — flavor-feature delta is intentional (Play policy); should be surfaced in the Settings UI (not in source). |

---

## Pass 5— API Contract Violations

| # | Location | Defect | Severity | Evidence Level | Action | Spec Reference |
|---|----------|--------|----------|----------------|--------|----------------|
| 5.1 | `composeApp/src/commonMain/.../ui/chat/ChatUiState.kt:265` and the OpenAI-compatible `FunctionCall.arguments: String` field (closes `proto-CF2`) | Kai's parse path— `SharedJson.parseToJsonElement(tc.arguments)` with `isLenient = true`. The OpenAI spec mandates `arguments` is a **JSON-stringified** object. Some downstream providers in Kai's chain (Gemini gateways, certain Anthropic-OpenAI-compat wrappers) return `arguments` as already-parsed JSON inline, which would deserialize as a JsonElement (object), not a String. With `isLenient = true`, the parse would happen successfully if the wire-form happens to be the same shape, but if a provider returns an unquoted JSON object Kotlin's lenient parser would still reject `{a: 1}`. The risk is **provider-specific**— Kai's tests in `commonTest/.../openaicompatible/` cover the canonical case; cross-provider robustness is uncovered. | medium | strong inference | port differently — when the field is encountered as a JsonElement (provider returned parsed JSON), accept it directly; only `parseToJsonElement` if it's a String. The `Tool.execute(args: Map<String, Any>)` interface should be fed the parsed Map either way. Closes `proto-CF2`. | Protocols §B2a (FunctionCall.arguments). |
| 5.2 | `composeApp/src/androidMain/.../MainActivity.kt:115-122` (closes `mech-CF2`) | `handleDeepLinkIntent` mutates `intent.removeExtra(EXTRA_OPEN_HEARTBEAT)` to dedup re-entry under configuration changes. Acceptance scenario #12 from the contracts phase ("the deep-link extra is consumed exactly once") relies on `DataRepository.requestOpenHeartbeat` being **idempotent**. There is no observable guard on the receiving end (the chat ViewModel observer just loads the heartbeat conversation; double-load would re-load the same conversation). The contract is therefore **idempotent-in-effect, not idempotent-by-design**. A future change to `requestOpenHeartbeat` that adds a side effect (e.g., logging "heartbeat opened" exactly once) would break the contract silently. | low | strong inference | port differently — make `requestOpenHeartbeat` explicitly debounced (StateFlow with distinctUntilChanged on a heartbeat-id key). Closes `mech-CF2`. | Contracts §Acceptance scenario 12. |
| 5.3 | `composeApp/src/commonMain/.../data/AppSettings.kt:importFromJson` and the absence of a top-level `version` field in the export (cross-references protocols §B5) | Settings export has no top-level `version` field. Combined with `ignoreUnknownKeys = true` (4.5), an export from a newer Kai (v2.0) imported into an older Kai (v1.0) **silently drops new sections**, and the import-success UI confirms "Imported N items". The user thinks their settings round-tripped but is missing v2.0-specific data on the v1.0 instance. | medium | observed fact | fix before porting — add `_schema_version: 1` (or `_kai_version`) at the top of every export; on import, refuse to load an export with a higher schema version than the running build supports, surfacing "this export was created by a newer version of Kai; please update first." | Protocols §B5. |
| 5.4 | `composeApp/src/androidMain/.../notifications/KaiNotificationListenerService.kt:onNotificationPosted` filter set | The class doc-comment lists the filter set— notifications-toggle off → drop, hard-blocked package → drop, FLAG_ONGOING_EVENT → drop, FLAG_FOREGROUND_SERVICE → drop, VISIBILITY_SECRET → drop. This is a **declared** filter contract. The actual code matches the doc comment. However the contract does **not** filter by `category` (e.g., `Notification.CATEGORY_REMINDER`, `CATEGORY_ALARM` could be considered transient); a future spec change might want this. | low | observed fact | leave behind — current filter matches doc-comment. | Architecture §Public Surfaces (notification listener). |
| 5.5 | `composeApp/src/androidMain/.../DaemonService.kt:44-47` (`onTimeout`) | `onTimeout` calls `stopForeground(STOP_FOREGROUND_REMOVE)` + `stopSelf()`. The protocols-phase SM-2 transition says "next foreground restores via MainActivity.onStart re-assertion." But if the user is **actively using** Kai when `onTimeout` fires (FGS time-limit on Android 14+), the activity is in the foreground and `onStart` won't re-fire until the user backgrounds and foregrounds the app. The user is left with no daemon during continued use — heartbeat misses go unnoticed. | low | strong inference | port differently — on `onTimeout`, post a "daemon paused" UI signal via DataRepository so the activity (if visible) can immediately re-assert. | Contracts §Focus subsystems → Foreground service. |
| 5.6 | `composeApp/src/desktopMain/.../tools/ShellCommandTool.kt:103` (`coerceIn(1, MAX_TIMEOUT_SECONDS)`) | The desktop `ShellCommandTool` exposes a `timeout` parameter to the model with a max of `MAX_TIMEOUT_SECONDS = 120L`. The Android sandbox has a different max (`ProotExecutor.kt:13` → `MAX_TIMEOUT_SECONDS = 180L`). A model that knows it can use 180s on Android would silently hit a 120s cap on desktop. Combined with mech-defect 6.2 (three sources of truth), this is a **user-facing parity violation**— the same shell tool behaves differently per platform. | medium | observed fact | port differently — lift the cap to a single `SandboxLimits.MAX_TIMEOUT_SECONDS` in commonMain and surface it to the model in the tool description. (See mech-defect 6.2.) | Architecture §Concurrency Model; contracts §Sandbox terminal. |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 4 |
| Medium | 7 |
| Low | 7 |
| **Total** | 18 |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 3. Concurrency and resources | 0 | 0 | 3 | 3 | 6 |
| 4. Security and trust | 0 | 4 | 2 | 1 | 7 |
| 5. API contract violations | 0 | 0 | 2 | 3 | 5 |

### Top Findings

1. **4.1 — `cleartextTrafficPermitted="true"` globally** (`network_security_config.xml`)— the entire Android app permits HTTP. **High**, **fix before porting**.
2. **4.4 — Notification content forwarded to cloud LLMs** without per-notification gating; potentially leaks 2FA codes / payment / message previews to third-party providers. **High**, **fix before porting**.
3. **4.2 — Desktop encryption key stored as plaintext bytes** (`EncryptedFileSettings.kt`); same-user malware reads the key trivially. **High**, **fix before porting**.
4. **4.3 — API keys and email passwords exported in cleartext** in the user-pickable JSON file; no warning UI. **High**, **fix before porting**.
5. **5.3 — No `_schema_version` in settings export** combined with `ignoreUnknownKeys=true` makes cross-version imports lose data silently. **Medium**, **fix before porting**.
6. **3.1 / 3.2 — `runBlocking { getString(...) }` in notification path; concurrent state writes in LinuxSandboxManager.** **Medium**, **fix before porting / port differently**.
7. **5.6 — Per-platform timeout cap drift** (desktop 120s, Android 180s) for the same model-facing tool. **Medium**, **port differently**.

### Carry-Forward Closure

| ID | Source Phase | Closed Because |
|----|--------------|---------------|
| mech-CF1 | defect-scan-mechanical | 3.1 — `runBlocking { getString(...) }` deadlock-risk semantic framing pinned. |
| mech-CF2 | defect-scan-mechanical | 5.2 — heartbeat-dedup contract ("idempotent-in-effect, not idempotent-by-design") pinned and routed for porting fix. |
| mech-CF3 | defect-scan-mechanical | 3.2 — concurrent `_state.value` writes in LinuxSandboxManager; recommended Mutex/actor pattern. |
| proto-CF1 | protocols | 4.5 — `ignoreUnknownKeys=true` on the shared `SharedJson` makes settings imports silently drop unknown keys. |
| proto-CF2 | protocols | 5.1 — `function.arguments` cross-provider drift (JSON-string vs parsed JSON) characterised. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | All three semantic passes (3, 4, 5) produced findings or documented "no defects found." | PASS | All three passes have findings (6 / 7 / 5). |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row in §Pass 3, §Pass 4, §Pass 5 has all four columns populated. |
| 3 | Pass 5 findings cite the contract or protocol reference they violate. | PASS | Each Pass 5 row carries a "Spec Reference" column with the phase and section anchor. |
| 4 | Findings are organized by pass and sorted by severity; summary tables match the detailed findings. | PASS | One table per pass; high before medium before low. Severity and per-pass counts match (4 high / 7 medium / 7 low / 18 total). |
| 5 | Any carry_forward entries that targeted defect-scan-semantic have been resolved or explicitly re-routed. | PASS | All 5 inherited carry_forward items closed in §Carry-Forward Closure. |
| 6 | Findings are marked with evidence levels. | PASS | Every finding has Evidence Level column. |

**Validated by:** 2026-05-06 (defect-scan-semantic, session 5)
**Overall:** PASS
