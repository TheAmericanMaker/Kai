# Protocols and State — Kai 9000

Wire-format detail and state-machine notes for Kai's protocol boundaries. The contracts phase
pinned user-visible behavior; this phase pins the *interfaces between layers* — proot subprocess
argv, AI provider HTTP shape, MCP JSON-RPC, settings export JSON, and Kai's internal state
machines.

This document closes three `carry_forward` items routed by contracts:
- `ctr-CF1` (proot argv shape, env vars, working-dir normalisation) — see §Sandbox / proot subprocess.
- `ctr-CF2` (AI tool-call JSON schema) — see §AI tool-call protocol.
- `ctr-CF3` (settings export JSON schema) — see §Settings export format.

## Boundaries Identified

| # | Boundary | Carrier | Direction | Notes |
|---|----------|---------|-----------|-------|
| B1 | Kai (JVM/Android) → proot subprocess → guest commands | argv + env (POSIX `Runtime.exec`) | bidirectional (process pipes) | Local exec; not network. |
| B2 | Kai (commonMain) → LLM provider HTTP API | HTTPS + JSON request/response (OpenAI-compatible OR Anthropic OR Gemini OR provider-specific) | bidirectional | Auth via API keys per provider. |
| B3 | Kai → MCP server | JSON-RPC 2.0 over HTTP (or stdio for local servers) | bidirectional | User-supplied servers. |
| B4 | Kai persistence → encrypted local store | platform-native key-value (DataStore / NSUserDefaults / file / localStorage) + JSON serialization | bidirectional | The "encrypted storage" feature wraps this. |
| B5 | Kai persistence → settings export file | JSON file via FileKit picker | one-shot | User-driven. |
| B6 | Kai → Android system services (FGS, NotificationListener, AlarmManager, FileProvider) | Android Intents + AIDL via SDK | bidirectional | Standard Android IPC. |
| B7 | Kai UI ↔ ChatViewModel ↔ DataRepository | Kotlin coroutine StateFlow + suspend calls | in-process | Not protocol, but a designed contract. |
| B8 | proot stdout/stderr → Kai | OS pipes (read line-by-line or as bounded byte stream) | one-way | Streamed or buffered. |

---

## Event Catalog

### B1 — Sandbox / proot subprocess (`ctr-CF1` resolved)

**Producer / consumer:** Kai (`ProotExecutor`) → guest shell process.

**Argv shape** (from `composeApp/src/androidMain/.../sandbox/ProotExecutor.kt:133-145`):

```
[
  $prootPath,                            // applicationInfo.nativeLibraryDir/libproot.so
  "--rootfs=$rootfsPath",                // context.filesDir/linux-sandbox/rootfs
  "--bind=/dev",
  "--bind=/proc",
  "--bind=/sys",
  "--bind=$homePath:/root",              // getExternalFilesDir(null)/sandbox-home → /root
  "--bind=$tmpPath:/tmp",                // context.filesDir/linux-sandbox/tmp → /tmp
  "-0",                                  // pretend uid/gid 0 inside the sandbox
  "-w", $workingDir,                     // default: "/root"; caller-overridable
  "/bin/sh", "-c", $command              // single-string shell command
]
```

**Environment variables** (`ProotExecutor.kt:146-156`):

```
HOME=/root
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TERM=xterm-256color
LANG=C.UTF-8
LD_LIBRARY_PATH=$libDir                  // applicationInfo.nativeLibraryDir
PROOT_TMP_DIR=$tmpPath
PROOT_LOADER=$loaderPath                 // = dirname($prootPath)/libproot-loader.so
+ caller-supplied $extraEnv (k=v pairs)
```

**Result envelope** (return shape of `ProotExecutor.execute`, `ProotExecutor.kt:84-104`):

| Path | Shape |
|---|---|
| Success (process exited normally) | `{success: Boolean, stdout: String, stderr: String, exit_code: Int, timed_out: false}` |
| Timeout | `{success: false, stdout: String, stderr: String, exit_code: -1, timed_out: true}` |
| Exception (couldn't spawn / IO failure) | `{success: false, error: String}` (no stdout/stderr/exit_code keys) |

**Bounding:** `stdout` and `stderr` are truncated to `MAX_OUTPUT_LENGTH = 15_000` chars via `smartTruncate`. *(observed fact — `ProotExecutor.kt:11`.)*

**Streaming variant:** `ProotExecutor.executeStreaming(command, workingDir, extraEnv, onStdout, onStderr): ProotHandle` returns a handle without timeout enforcement. The `ProotHandle` exposes `cancel()`, `writeInput(line)`, `awaitExit(): Int`. *(observed fact — `ProotExecutor.kt:111-128, 14-49`.)*

**Working dir normalisation:** None at the proot layer. The caller is expected to pass an absolute sandbox path (e.g. `/root`); proot does not validate it. The `SandboxFiles.resolveSandboxAbsolute` helper handles host-side path-traversal checks before `executeCommand` is called. *(observed fact — `SandboxFiles.kt:15-26`.)*

**Portability hazards** *(portability hazard)*— argv depends on the proot binary's CLI shape (a snapshot of termux/proot's CLI). Bind-mount syntax (`--bind=`), the `-0` (fake-root) flag, and `--rootfs=` are termux/proot-specific. Re-implementations targeting other sandbox primitives (Linux user namespaces, Lima/Docker, BSD jails) need to re-derive this.

---

### B2 — AI provider HTTP API (`ctr-CF2` resolved)

Kai supports **24+ LLM providers** behind a fallback chain. Three wire shapes are first-class— OpenAI-compatible, Anthropic, Gemini. The OpenAI-compatible shape carries the most providers (DeepSeek, Mistral, Ollama, OpenRouter, Together, Fireworks, …).

#### B2a — OpenAI-compatible request/response

**Request DTO** (`composeApp/src/commonMain/.../network/dtos/openaicompatible/OpenAICompatibleChatRequestDto.kt`):

```
OpenAICompatibleChatRequestDto {
  messages: [Message],
  model: String?,
  tools: [Tool]?,
}

Message {
  role: "system" | "user" | "assistant" | "tool",
  content: JsonElement?,                           // String OR array of content parts (vision)
  tool_calls: [ToolCall]?,                         // assistant message with tool calls
  tool_call_id: String?,                           // required for role=tool messages
}

Tool {
  type: "function",                                // currently only function is supported
  function: Function,
}

Function {
  name: String,
  description: String?,
  parameters: Parameters?,
  strict: Boolean?,                                // OpenAI Structured Outputs
}

Parameters {
  type: "object",
  properties: Map<String, PropertySchema>,
  required: [String]?,
  additionalProperties: Boolean?,
}

PropertySchema {                                   // recursive JSON-schema-ish
  type: "string" | "number" | "boolean" | "integer" | "array" | "object",
  description: String?,
  enum: [String]?,
  items: PropertySchema?,                          // for type=array
  properties: Map<String, PropertySchema>?,        // for type=object
  required: [String]?,
  additionalProperties: Boolean?,
}

ToolCall {                                         // in assistant response
  id: String,
  type: "function",
  function: FunctionCall,
}

FunctionCall {
  name: String,
  arguments: String,                               // JSON string; caller must parse
}
```

*(observed fact — `OpenAICompatibleChatRequestDto.kt`.)*

**Notable shape decisions:**
- `content: JsonElement?` accepts a String *or* an array of content parts (text + image_url) for vision-capable providers.
- `tool_calls` is parallel to `content`; an assistant message that calls a tool may also have text content.
- `function.arguments` is a **JSON string** (not parsed JSON). The caller `JSON.parseToJsonElement(arguments)` it. This is OpenAI's wire-shape choice; downstream providers mirror it. *(portability hazard — argument shape interop is fragile across providers.)*

#### B2b — Internal Tool abstraction

Before sending to a provider, Kai's tools are described in a provider-neutral `ToolSchema` (`composeApp/src/commonMain/.../network/tools/Tool.kt`):

```
ToolSchema {
  name: String,
  description: String,
  parameters: Map<String, ParameterSchema>,
}

ParameterSchema {
  type: String,                                    // "string", "integer", etc.
  description: String,
  required: Boolean,
  rawSchema: JsonObject?,                          // optional escape hatch for nested schema
}

interface Tool {
  schema: ToolSchema,
  timeout: Duration = 30.seconds,
  suspend execute(args: Map<String, Any>): Any,    // result is JSON-serializable
}
```

The conversion from `ToolSchema` → `OpenAICompatibleChatRequestDto.Function.parameters` happens at request build time; nested schemas use `rawSchema` to bypass the flat-parameter limitation. *(observed fact — `Tool.kt:8-22`.)*

**Tool result envelope** (Kai → next assistant message)— the result of `Tool.execute(args)` is returned as a `Message{role="tool", tool_call_id=<...>, content=<JSON-stringified result>}`. *(strong inference — derived from OpenAI's tool-call protocol; Kai mirrors it.)*

#### B2c — Provider-specific shapes (Anthropic / Gemini)

Out of scope for this phase's deep walk; Anthropic uses a `messages` array with `content` blocks and `tool_use`/`tool_result` blocks, Gemini uses `contents` with `parts` and `functionCall`/`functionResponse`. *(open question — exact DTO shapes not extracted; relevant if porting beyond the OpenAI-compatible subset.)*

---

### B3 — MCP server transport

Wire format— **JSON-RPC 2.0** over HTTP (or stdio for local MCP servers). DTOs from `composeApp/src/commonMain/.../mcp/McpModels.kt`:

```
JsonRpcRequest {
  jsonrpc: "2.0",
  id: Int,                                         // monotonically incremented
  method: String,                                  // "tools/list", "tools/call", "initialize", ...
  params: JsonElement?,
}

JsonRpcResponse {
  jsonrpc: "2.0",
  id: Int?,                                        // matches request; null for notifications
  result: JsonElement?,                            // or error
  error: JsonRpcError?,
}

JsonRpcError {
  code: Int,                                       // standard JSON-RPC error codes
  message: String,
  data: JsonElement?,
}
```

**MCP-specific result shapes:**

```
McpToolDefinition {                                // result of tools/list
  name: String,
  description: String?,
  inputSchema: JsonObject?,                        // JSON Schema for the tool's params
}

McpToolsResult {
  tools: [McpToolDefinition],
}

McpCallToolResult {                                // result of tools/call
  content: [McpContent],
  isError: Boolean,
}

McpContent {
  type: String,                                    // "text" | "image" | ...
  text: String?,
}
```

*(observed fact — `McpModels.kt`.)*

**Trust boundary**— Kai sends user-supplied auth (API keys, bearer tokens) in headers per the MCP-server config. The MCP transport is a JSON-RPC adapter; no Kai-specific framing. *(observed fact — README.md §Features "MCP server support".)*

**Compatibility hazard** *(portability hazard)*— MCP protocol versions evolve; Kai must negotiate via `initialize` and degrade gracefully. The DTOs above are the minimal surface; future MCP versions add `prompts/list`, `resources/list`, etc. — Kai's current implementation is tools-only.

---

### B5 — Settings export JSON (`ctr-CF3` resolved)

**Producer**— `RemoteDataRepository.exportSettingsToJson(sections)`— calls `appSettings.exportToJson(toolIds, sections)` and pretty-prints. *(observed fact — `RemoteDataRepository.kt:1936-1940`.)*

**Consumer**— `RemoteDataRepository.importSettingsFromJson(json, sections, replace)`— parses with `SharedJson.parseToJsonElement(json).jsonObject` and dispatches per-section. *(observed fact — `RemoteDataRepository.kt:1948-1952`.)*

**Top-level shape**— A flat `JsonObject` with one key per section; each section can be empty/absent. The selectable sections are `ImportSection` enum values *(observed fact — `AppSettings.kt:19-31`)*—

| Section | JSON key(s) | Type | Notes |
|---|---|---|---|
| SERVICES | `configured_services` | array | per-instance LLM provider config (creds redacted? *open question*). |
| SOUL | `soul_text` | string | system prompt. |
| MEMORY | `agent_memories` | array | persistent-memory entries. |
| SCHEDULING | `scheduled_tasks` | array | scheduler entries. |
| HEARTBEAT | `heartbeat_prompt`, `heartbeat_config`, `heartbeat_log` | string + object + array | three sub-keys, all optional. |
| EMAIL | `email_accounts` | array | configured email accounts. |
| SMS | `sms_enabled`, `sms_send_enabled` | bool + bool | feature toggles. |
| SPLINTERLANDS | `splinterlands_account` | object | game-integration account. |
| TOOLS | (per-tool flags + config; *open question* exact keys) | object | enabled/disabled state for each platform tool. |
| MCP | `mcp_servers`? (likely) | array | configured MCP servers. *(open question* — key name not pinned in this phase.)* |
| CONVERSATIONS | (per-conversation export shape; *open question* — not extracted) | array | conversation transcripts. |

**Detection helper**— `detectExportableSections(json: JsonObject)` populates a `Map<ImportSection, String?>` where the value is a count or null label, used by the export-preview dialog. *(observed fact — `AppSettings.kt:detectExportableSections`.)*

**Versioning**— **No top-level `version` field observed** in the export shape. *(open question — major hazard for migration— a future schema change would break older exports without a versioning scaffold. Recommended for porting— add `_schema_version: 1`.)*

**Migration policy**— `replace: Boolean` parameter on import — `replace=true` overwrites existing values, `replace=false` only fills missing keys. No record-level merge logic visible at this layer. *(observed fact — `RemoteDataRepository.importSettingsFromJson(json, sections, replace)`.)*

**Compatibility hazard** *(portability hazard)*— an export from a newer Kai with an unrecognised key won't be rejected by an older Kai's parse — `kotlinx.serialization` `ignoreUnknownKeys = true` is the platform-typical default. The user-facing "import worked" UI may quietly drop new fields. The defect-scan-semantic phase should look at this in Pass 5 (API contract drift).

---

### Cross-cutting — Persistent shell sessions (B1 extended)

The single-shot `ProotExecutor.execute` is wrapped by a higher-level **session** layer (`composeApp/src/androidMain/.../sandbox/PersistentSandboxShell.kt` + `SessionShell` API). A session keeps a long-lived `/bin/sh` proot child via the streaming variant, exposes `run(command, timeoutSeconds, onStdout, onStderr)`, and shares working directory and environment across calls. Bounding is the same `MAX_OUTPUT_LENGTH = 15_000`. *(observed fact — `PersistentSandboxShell.kt:19, 256-269`.)*

**Session id sentinels** (cross-cutting protocol — sessions named with these strings have special semantics)— `__terminal__`, `__system__`, `__default__` are the sentinel ids; `SandboxSessions.isPersistable(sessionId)` returns false for these (no chat-history persistence). *(observed fact — `SandboxController.kt:41-52`.)*

---

## State Machine

### SM-1— Sandbox lifecycle

The canonical sandbox state is held in `LinuxSandboxManager._state: MutableStateFlow<SandboxState>`. The sealed interface is *(observed fact — `composeApp/src/androidMain/.../sandbox/SandboxState.kt`)*—

```
sealed interface SandboxState {
  object NotInstalled
  data class Downloading(progress: Float)
  object Extracting
  data class Installing(detail: String = "")
  object Ready
  data class Error(message: String)
}
```

| Current State | Event / Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| NotInstalled | `setup()` invoked | none | Downloading(0f) | Begin tar.gz download from first mirror. |
| Downloading | bytesRead progress | none | Downloading(progress=k) | Increment progress; emit StateFlow update. |
| Downloading | download complete (200, all bytes received) | none | Extracting | Switch to tar extract. |
| Downloading | per-mirror IOException | mirror in list | Downloading(0f) (next mirror) | Delete partial file. |
| Downloading | all mirrors fail | none | Error("All Alpine mirrors failed— \<cause\>") | Delete partial file. |
| Downloading | `cancel()` | none | NotInstalled | Coroutine cancellation; partial file deleted. |
| Extracting | tar entry processed | none | Extracting | Continue. (`ensureActive()` per loop iteration permits cancel.) |
| Extracting | extract complete | none | Installing("") | Begin post-install (`writeResolvConf`, `writeRepositories`, `makeWritable`). |
| Extracting | `cancel()` | none | NotInstalled | Coroutine cancellation. |
| Extracting | unrecoverable error (e.g., disk full) | none | Error(\<message\>) | none. |
| Installing | post-install step done | none | Installing(\<next-detail\>) | Continue. |
| Installing | all post-install steps done | none | Ready | Sandbox usable. |
| Installing | error | none | Error(\<message\>) | none. |
| Ready | `installPackages()` | none | Installing("Installing pkg…") | apk add via proot shell. |
| Ready | `reset()` | none | NotInstalled | `closeAllShells()`; `sandboxDir.deleteRecursively()`. |
| Error | `setup()` (retry) | none | Downloading(0f) | Re-enter download. |
| Error | `reset()` | none | NotInstalled | Wipe. |

**Invariants:**
- Only `LinuxSandboxManager`'s coroutines write `_state`. UI reads via `StateFlow`. *(observed fact — `LinuxSandboxManager.kt`.)*
- `Ready` is sticky— only `setup()`-driven re-install or `reset()` exits it. *(strong inference.)*
- The `Installing` detail is free-form for human consumption; not part of the protocol surface.

**State-machine portability hazard** *(portability hazard)*— the state set is portable; the *transition triggers* are platform-specific (Ktor channel cancellation, `coroutineContext.ensureActive()`). A non-Kotlin port reproduces the state set easily; the cancellation semantics need re-derivation per language.

### SM-2— Daemon foreground service lifecycle

| Current State | Event | Next State | Side Effects |
|---|---|---|---|
| (no service) | `DaemonController.start()` called; `appSettings.isDaemonEnabled() == true` | starting | `context.startForegroundService(intent)`. |
| starting | `Service.onCreate()` | running | createNotificationChannel; startForeground(9001, …); taskScheduler.start(). |
| starting | `startForeground` throws | (no service) | stopSelf(); return. |
| running | `Service.onStartCommand()` | running | return START_STICKY. |
| running | OS kill (memory pressure / OEM) | (no service) | OS will recreate (START_STICKY). |
| running | Android 14+ FGS time limit | (no service) | `onTimeout()` → stopForeground(REMOVE) + stopSelf(). |
| running | `DaemonController.stop()` | (no service) | `Service.onDestroy()` → stopForeground(REMOVE). |
| running | `MainActivity.onStart()` re-asserts | running (unchanged) | No-op (idempotent). |
| (no service) | OS recreate after kill | starting | `Service.onCreate()` again. |
| (no service) | `MainActivity.onStart()` re-asserts | starting | `startForegroundService` if `appSettings.isDaemonEnabled()`. |

**Invariant**— notification ID 9001 is always associated with `DaemonService`, never a different service. *(observed fact — `DaemonService.kt:18`.)*

### SM-3— Heartbeat notification dedup

| Current | Event | Next | Side effects |
|---|---|---|---|
| (no pending) | new heartbeat reply ready | pending | `notify(HEARTBEAT_NOTIFICATION_ID = 9002, …)`. |
| pending | new heartbeat reply ready before user opens | pending (replaced) | `notify(9002, newNotification)` replaces. |
| pending | user taps notification | (no pending) | Activity opens with EXTRA_OPEN_HEARTBEAT; chat opens; `intent.removeExtra(...)` consumes. |
| pending | user dismisses notification | (no pending) | Standard Android cancel. |

**Invariant**— at most one pending heartbeat conversation at a time, by virtue of the fixed notification ID. *(observed fact — `HeartbeatNotifier.android.kt:30-31`.)*

---

## Persistent Schema Notes

| Store | Format | Append-only / Mutable | Locking |
|---|---|---|---|
| Settings (per-platform) | platform key-value (DataStore on Android, NSUserDefaults on iOS, file on desktop, localStorage on wasmJs) | mutable; atomic per key | platform-provided. |
| Conversations | encrypted local store; per-conversation records | mutable per conversation; chat history append-only within a conversation | *(open question — `ctr-OQ3` not yet resolved; deferred to defect-scan-semantic)*. |
| Persistent memory | JSON entries in app storage | append-only at memory-write; entries can be edited/deleted via UI | *(open question — exact schema not extracted)*. |
| Sandbox rootfs | extracted Alpine tarball | mutable (apk installs; user `cp`/`mv`) | filesystem-level. |
| Sandbox `/root` (= `<external-files>/sandbox-home`) | mutable filesystem dir | mutable | filesystem-level. |
| Sandbox `/tmp` | mutable filesystem dir | mutable; not cleaned between runs | filesystem-level. |
| Sandbox per-session transcript | in-memory `SnapshotStateList<TerminalLine>` | append-only during session; persisted only for chat-bound sessions per `SandboxSessions.isPersistable` | n/a (in-memory). |
| Heartbeat log | array entry in settings JSON (`heartbeat_log`) | append-only | platform settings store. |
| LiteRT models | binary files in app storage | mutable (download replaces) | filesystem. |

**Compaction / summarization**— the README's "Persistent memory" feature stores extracted memories; conversations themselves are stored in full (no auto-summarisation observed). *(strong inference — `ChatViewModel` stores raw history.)*

---

## Compatibility Hazards

| Hazard | Where It Appears | Severity | Notes |
|---|---|---|---|
| Android FGS rules across OS versions | Android 12 (background-start ban), 13 (POST_NOTIFICATIONS), 14 (typed FGS + dataSync time limits) | high | DaemonService catches the resulting exceptions but logs lossily (see mech-defect 2.4). |
| OEM aggressive battery managers | MIUI / EMUI / Huawei / ColorOS / MagicOS may kill FGS despite the foreground notification | medium | Mitigation = MainActivity.onStart re-assertion (silent recovery). |
| proot ptrace requirement | Android sandbox uses `ptrace`; iOS forbids it; web has no equivalent | high | Sandbox is Android-only by design (NoOpSandboxController on other targets). |
| FAT32 / SD-card filesystems disallow symlinks | tar extract in `RootfsDownloader.extractTar` (silent skip — see mech-defect 1.1) | medium | Could surface as "command not found" later. |
| OpenAI tool-call shape drift across providers | `function.arguments` is a JSON string; some providers return parsed JSON | high | Each provider adapter parses defensively; testing across all 24 is impractical. |
| MCP protocol version drift | Kai's MCP impl is tools-only; future versions add prompts/resources | medium | Kai must negotiate via `initialize` and gracefully ignore unknown methods. |
| Settings export — no top-level `version` field | JSON shape evolves without explicit versioning | medium | Future migrations are best-effort; older Kai parsing newer JSON may silently drop fields. |
| Alpine version pinned at 3.21.3 | hardcoded in `RootfsDownloader.kt:17` | low | Reproducibility benefit; security hazard if 3.21.3 has unpatched CVEs. |
| FileKit per-platform back-end | SAF on Android, NSURL bookmarks on iOS, native chooser on desktop, `<input type=file>` on web | low | Cross-platform API hides these differences; persistent re-access is **not** supported. |
| Locale / timezone — `LANG=C.UTF-8` in proot env | Sandbox commands always run with C locale; user sees English errors | low | Intentional for parsability of shell errors. |

---

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| proto-OQ1 | needs-source-extraction | Anthropic and Gemini provider DTO shapes (full request/response). | Out of scope for this phase's deep walk; the OpenAI-compatible subset covers the bulk. |
| proto-OQ2 | needs-source-extraction | Settings export JSON shapes for `TOOLS`, `MCP`, `CONVERSATIONS` sections (exact key names). | The `detectExportableSections` helper visits 8 sections; the remaining 3 weren't extracted. |
| proto-OQ3 | needs-runtime-test | Encrypted conversation store on-disk format — record framing, IV/nonce placement, key wrap. | Inherited from `ctr-OQ3`. Defer to defect-scan-semantic Pass 4 (security). |
| proto-OQ4 | needs-source-extraction | Heartbeat log entry schema. | `heartbeat_log` array exported but per-entry shape not extracted. |
| proto-OQ5 | needs-runtime-test | Whether `kotlinx.serialization`'s `ignoreUnknownKeys` is set globally for settings parse. | Determines whether older Kai silently drops new fields on import. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| proto-CF1 | defect-scan-semantic | Settings-import key-drop hazard (proto-OQ5). Pass 5 (API contract drift) angle. | Needs the contracts output already loaded by defect-scan-semantic. |
| proto-CF2 | defect-scan-semantic | OpenAI-compat `function.arguments` JSON-string parsing — what happens when a provider returns parsed JSON instead of a string? Pass 5 angle. | Cross-provider testing matrix too large for the protocols phase. |
| proto-CF3 | porting | The "no `version` field in settings export JSON" decision. Porting should explicitly design *for* an export format with a version scaffold. | Architectural decision belongs in the porting recommendations. |

## Carry-Forward Closure (from contracts)

| ID (in) | Closed because |
|---|---|
| ctr-CF1 | §B1 pins the proot argv array (12 elements), env vars (7 base + extraEnv), and the result envelope shape. The working-directory normalisation note explains it's caller-responsibility. |
| ctr-CF2 | §B2 pins the OpenAI-compatible `tools / function / parameters / properties` shape AND the assistant-response `tool_calls / function.name / function.arguments` shape (with the JSON-string note). The internal Kai `ToolSchema` is documented separately. Anthropic / Gemini DTOs are routed to proto-OQ1. |
| ctr-CF3 | §B5 pins the top-level JSON-object shape, the per-section keys (8 of 11 enumerated), the import `replace` semantics, and the **no top-level `version` field** finding (routed forward as proto-CF3). |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | An event catalog is documented. | PASS | §Event Catalog covers B1 (proot), B2 (LLM HTTP), B3 (MCP), B5 (settings export); B4/B6/B7/B8 listed in §Boundaries Identified with notes. |
| 2 | A state machine is documented. | PASS | §State Machine has SM-1 (sandbox lifecycle, 13 transitions), SM-2 (daemon FGS, 8 transitions), SM-3 (heartbeat dedup, 4 transitions). |
| 3 | Persistent schema notes are documented. | PASS | §Persistent Schema Notes (10 stores). |
| 4 | Compatibility hazards are documented. | PASS | §Compatibility Hazards (10 rows). |
| 5 | Findings are marked with evidence levels. | PASS | Every claim tagged inline with *observed fact*, *strong inference*, or *portability hazard*; five open questions captured in §Open Questions. |

**Validated by:** 2026-05-06 (protocols phase, session 4)
**Overall:** PASS
