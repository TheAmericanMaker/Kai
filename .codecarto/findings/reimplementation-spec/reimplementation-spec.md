# Reimplementation Spec — Kai 9000

Final language-agnostic reimplementation plan and acceptance spec for Kai 9000. Synthesises every prior phase. The orchestrator chose **language-agnostic** mode at the strategic-alignment hook (skipping the opinionated variant). This spec is the canonical reference an independent team can use to build a Kai-equivalent on any stack.

This document closes the two inbound `carry_forward` items—
- `port-CF1` (four HIGH-security cluster) — see §Required Behaviors → §Security baseline and §Implementation Sequence row v0.
- `port-CF2` (Android-only-sandbox asymmetry) — see §External Dependencies → "Sandbox primitive" row and §Deliberate Non-Goals.

---

## System Summary

A multi-platform AI assistant that (a) wraps a fallback-chain of LLM provider HTTP APIs behind a unified chat interface, (b) persists conversations and a structured "persistent memory" locally with at-rest encryption, (c) supports user-supplied Model Context Protocol servers as an additional tool transport, (d) runs an autonomous-heartbeat scheduler that periodically prompts the LLM to surface follow-ups, and (e) on at least one platform embeds a real Linux user-land that the LLM can shell into via a tool.

Identity statement— *"one shared chat-and-tool agent core, with platform-specific adapters for sandbox, daemon, and notification surfaces; the platform with the deepest adapter is the platform that gives the user the most autonomy."*

Per the Kai 9000 source, that platform is Android — but the reimplementation is **not bound to that asymmetry** (see port-CF2 in §Deliberate Non-Goals).

---

## Conceptual Module Model

Modules listed in dependency-from-leaf-to-root order. Names are concept-level; the port can choose its own.

### `agent-core`

| Field | Value |
|---|---|
| **Responsibility** | Orchestrate a chat session — accept user input, route through the LLM provider chain, dispatch tool calls, maintain conversation state. |
| **Public inputs** | `userMessage`, `imageAttachments[]`, `toolCallResults[]`, `systemPrompt` ("soul"), `providerChain[]`, `availableTools[]`. |
| **Public outputs** | Streamed assistant text, structured tool-call requests, conversation transcript updates, persistent-memory write events. |
| **Owned state** | Active conversation transcript (in-memory; persisted by `chat-storage`); current provider index in fallback chain; in-flight tool calls. |
| **Invariants** | (i) Every assistant message either has text content OR tool-call requests OR both; (ii) Tool calls always receive a tool-result message before the next user message; (iii) Provider fallback is automatic on transport failure but never on tool-execution failure. |
| **Collaborators** | `provider-chain`, `tool-registry`, `chat-storage`, `memory-store`. |

### `provider-chain`

| Field | Value |
|---|---|
| **Responsibility** | Send chat requests to the user's configured LLM provider list, fall over on transport failures. Handle three wire shapes (OpenAI-compatible, Anthropic, Gemini) plus on-device LLM. |
| **Public inputs** | Chat messages with role/content/tool_calls; tool definitions (provider-neutral `ToolSchema`); provider credentials (per-instance api keys); provider chain order. |
| **Public outputs** | Streamed assistant tokens, completed assistant message, tool-call requests parsed from the response. |
| **Owned state** | Per-instance credential cache; in-flight HTTP connection; the canonical message history transformed for each provider. |
| **Invariants** | (i) Provider-neutral `ToolSchema` round-trips through every wire shape losslessly for the supported subset; (ii) `function.arguments` is accepted as either String OR parsed JSON object (defect 5.1 fix); (iii) Cleartext HTTP is OFF by default; per-host opt-in only (defect 4.1 fix). |
| **Collaborators** | `agent-core`, `tool-registry`. |

### `tool-registry`

| Field | Value |
|---|---|
| **Responsibility** | Catalogue available tools, dispatch tool-call requests to the correct executor, return structured tool-results. |
| **Public inputs** | `ToolSchema` registrations (built-in tools + MCP-server tools), tool-call requests `{name, arguments: parsed-object, id}`. |
| **Public outputs** | Tool results as JSON-serializable values, with success/error envelope. |
| **Owned state** | Registered tools by name; per-tool timeout. |
| **Invariants** | (i) Every registered tool has a non-empty schema; (ii) Tool execution never blocks the main thread; (iii) Tool-result content has a stable shape `{role: "tool", tool_call_id, content: <jsonified-result>}`. |
| **Collaborators** | `provider-chain`, `mcp-client`, `sandbox-shell-tool`, `notification-tool`, `web-search-tool`, `calendar-tool`, etc. |

### `mcp-client`

| Field | Value |
|---|---|
| **Responsibility** | Speak JSON-RPC 2.0 to user-configured MCP servers (HTTP or stdio). Surface MCP tools into `tool-registry`. |
| **Public inputs** | MCP server config `{id, transport, url, auth-headers}`, JSON-RPC method/params. |
| **Public outputs** | Discovered tool list (`tools/list`), tool-call results (`tools/call`). |
| **Owned state** | Per-server JSON-RPC id sequence; cached tools list. |
| **Invariants** | (i) Negotiates protocol version via `initialize`; (ii) Gracefully ignores unknown methods/result fields. |
| **Collaborators** | `tool-registry`. |

### `sandbox`

| Field | Value |
|---|---|
| **Responsibility** | Provide a sandboxed Linux command-execution surface to the LLM. State machine `NotInstalled → Downloading(progress) → Extracting → Installing(detail) → Ready / Error(message)`. |
| **Public inputs** | `setup()`, `cancel()`, `reset()`, `installPackages()`, `executeCommand(command, sessionId, workingDir, env)— Result`, `executeCommandStreaming(command, sessionId, workingDir, env, onStdout, onStderr)— Handle`. |
| **Public outputs** | Sandbox state stream (StateFlow-equivalent), command result envelope `{success, stdout, stderr, exit_code, timed_out}`. |
| **Owned state** | Sandbox rootfs on disk; per-session shell processes; sandbox limits (output cap, default/max timeout). |
| **Invariants** | (i) Path traversal protected (no `..` escapes from sandbox roots); (ii) State transitions serialised through a single mutator (Mutex/actor); (iii) Sandbox limits are a single source of truth (`SandboxLimits` constants in core, per defect 5.6/6.2 fix); (iv) Tar extraction logs per-entry symlink failures and surfaces them in post-extract state (defect 1.1 fix); (v) `Ready` is reached only after every post-install step succeeded (defect 1.2 fix). |
| **Collaborators** | `tool-registry` (sandbox-shell-tool); platform-specific sandbox-primitive adapter (see §External Dependencies → "Sandbox primitive"). |

### `chat-storage`

| Field | Value |
|---|---|
| **Responsibility** | Persist conversations (encrypted at rest) and persistent-memory entries. |
| **Public inputs** | Conversation create/update/delete; memory entry create/update/delete; export/import (per `settings-export`). |
| **Public outputs** | Loaded conversations + memories; export JSON. |
| **Owned state** | On-disk encrypted records; per-platform encryption-key handle. |
| **Invariants** | (i) Encryption key is held in the platform's secure-enclave equivalent — Keystore on Android, Keychain on macOS, DPAPI on Windows, libsecret on Linux. **Never as a plaintext file** (defect 4.2 fix); (ii) Migration from an older store is transactional (write-new → fsync → delete-old; defect 4.6 fix); (iii) Conversation persistence is encrypted at the record level (algorithm and key-wrap exposed in §Protocols and Persisted State). |
| **Collaborators** | `agent-core` (writes), settings UI (reads), `settings-export`. |

### `memory-store`

| Field | Value |
|---|---|
| **Responsibility** | Store and retrieve "persistent memory" entries the LLM writes via tool. |
| **Public inputs** | `addMemory(content, tags, ts)`, `removeMemory(id)`, `listMemories()`. |
| **Public outputs** | Memory list ordered by recency; matched memories for a given query (similarity or substring). |
| **Owned state** | On-disk memory entries (encrypted via `chat-storage`'s key). |
| **Invariants** | (i) Memory writes are append-only at the record level; deletions are tombstoned; (ii) Memories are exportable / importable per `ImportSection.MEMORY`. |
| **Collaborators** | `chat-storage`, `agent-core`. |

### `daemon` (platform-specific surface)

| Field | Value |
|---|---|
| **Responsibility** | Keep the app process alive on platforms that allow it, so the heartbeat scheduler can fire while the UI is backgrounded. |
| **Public inputs** | `start()`, `stop()`, daemon-enabled-flag from settings. |
| **Public outputs** | Daemon-running state stream; "daemon paused" UI signal on platform-mandated stop (e.g., Android FGS time-limit; defect 5.5 fix). |
| **Owned state** | Foreground-service binding (Android), Background-task identifier (iOS), system-tray entry (desktop), service-worker registration (web). |
| **Invariants** | (i) On platforms without a long-lived background surface, daemon is a no-op and the heartbeat only runs while the app is open; (ii) On Android, daemon is auto-restarted via `START_STICKY` AND re-asserted on every UI foreground (idempotent); (iii) Errors are logged with their actual exception type, not swallowed under generic `Exception` (defect 2.3/2.4 fix); (iv) On `onTimeout`, post a UI signal so a foregrounded app can re-assert immediately (defect 5.5 fix). |
| **Collaborators** | `heartbeat-scheduler`, platform OS APIs (FGS / BGTaskScheduler / system-tray / service-worker). |

### `heartbeat-scheduler`

| Field | Value |
|---|---|
| **Responsibility** | Periodically run a self-check prompt against the LLM, surfacing follow-ups via notifications. Configurable interval. |
| **Public inputs** | Heartbeat config (interval, prompt template, dedup policy), notification-store snapshot. |
| **Public outputs** | New conversation entries; notification post (one pending at a time, replace-on-new). |
| **Owned state** | Last-fire timestamp; pending-notification id. |
| **Invariants** | (i) At most one pending heartbeat conversation at a time (single fixed notification id); (ii) Deep-link consumption is idempotent-by-design (debounced StateFlow keyed by heartbeat id; defect 5.2 fix); (iii) **MUST NOT forward third-party notification content to off-device LLM providers without explicit per-app opt-in** (defect 4.4 fix). |
| **Collaborators** | `daemon`, `agent-core`, `notification-store`. |

### `notification-store` (platform-specific surface)

| Field | Value |
|---|---|
| **Responsibility** | On platforms with a system notification listener (Android FOSS flavor only in Kai 9000), capture other apps' notifications for use as heartbeat context. |
| **Public inputs** | System notification events (filtered— respect FLAG_ONGOING_EVENT, FLAG_FOREGROUND_SERVICE, VISIBILITY_SECRET, hard-blocked packages). |
| **Public outputs** | NotificationRecord list; sync-state stream. |
| **Owned state** | Per-record store; sync state. |
| **Invariants** | (i) **MUST NOT auto-export to LLM** without a user-configured per-app allowlist for the LLM-forward path (defect 4.4 fix); (ii) Listener-callback work is offloaded from the system-bound binder thread (defect 3.4 fix); (iii) Filter set is documented and matches code. |
| **Collaborators** | `heartbeat-scheduler`, `tool-registry` (notification-lookup tools). |

### `settings-store`

| Field | Value |
|---|---|
| **Responsibility** | Persist all user-configurable values (provider chain, soul, daemon-enabled, heartbeat config, MCP servers, etc.) in a per-platform encrypted key-value store. |
| **Public inputs** | get/set per key; bulk export/import (`settings-export`). |
| **Public outputs** | Per-key values; export/import handlers. |
| **Owned state** | Per-platform store handle. |
| **Invariants** | (i) Concurrent access is serialised (Mutex or ConcurrentHashMap; defect 3.3 fix); (ii) Settings parser is **strict** (no `ignoreUnknownKeys=true` for settings; defect 4.5 fix); unknown keys surface a user-visible warning. |
| **Collaborators** | every other module that reads config. |

### `settings-export`

| Field | Value |
|---|---|
| **Responsibility** | Round-trip the user's full configuration as a JSON file. |
| **Public inputs** | `exportToJson(sections, mode— full | sanitised)`, `importFromJson(json, sections, replace)`. |
| **Public outputs** | JSON file with `_schema_version: 1`; per-section keys per `ImportSection` enum. |
| **Owned state** | Schema version constant. |
| **Invariants** | (i) **`_schema_version` field is mandatory** on every export; on import, refuse newer-than-current versions with a clear error (defect 5.3 fix; closes proto-CF3); (ii) **Sanitised mode** omits secrets (api keys, email passwords) and the importer prompts for them (defect 4.3 fix); (iii) Import is **transactional**— on partial parse failure, no settings are mutated. |
| **Collaborators** | `settings-store`, file picker. |

### `ui-shell`

| Field | Value |
|---|---|
| **Responsibility** | Render Compose-Multiplatform-equivalent UI for chat, settings, memory, sandbox terminal, heartbeat history. |
| **Public inputs** | ViewModels' StateFlows. |
| **Public outputs** | Rendered UI; user input events. |
| **Owned state** | Composition state, navigation state. |
| **Invariants** | (i) UI is decoupled from platform-specific I/O; (ii) Resource string lookups are **not** done synchronously inside callback paths (defect 3.1 fix). |
| **Collaborators** | every ViewModel. |

---

## Layer Split

| Module | Layer |
|---|---|
| `agent-core` | core semantics |
| `provider-chain` | core semantics + adapters (HTTP wire shapes are adapters) |
| `tool-registry` | core semantics |
| `mcp-client` | adapter (JSON-RPC over HTTP/stdio) |
| `sandbox` | core semantics (interface) + adapter (per-platform sandbox primitive) |
| `chat-storage` | core semantics + adapter (per-platform encrypted store) |
| `memory-store` | core semantics |
| `daemon` | adapter (per-platform background-execution model) |
| `heartbeat-scheduler` | core semantics |
| `notification-store` | adapter (per-platform listener) |
| `settings-store` | core semantics + adapter (per-platform secure key-value store) |
| `settings-export` | core semantics |
| `ui-shell` | delivery surface |
| Platform shells (Android app, iOS app, Desktop app, Web app) | delivery surfaces |

---

## Required Behaviors

### Chat
- **MUST** support 24+ LLM providers via three primary wire shapes (OpenAI-compatible, Anthropic, Gemini) plus on-device.
- **MUST** automatically fall over to the next provider on transport failure; tool-execution failure surfaces to the LLM as a tool-result message and **MUST NOT** auto-fall-over.
- **MUST** stream assistant tokens.
- **MUST** persist conversations encrypted at rest.

### Tools
- **MUST** register a provider-neutral `ToolSchema` and translate to each wire shape at request time.
- **MUST** accept `function.arguments` as either a JSON string or a parsed JSON object (provider-drift tolerance).
- **MUST** dispatch tool-call results back to the LLM as `{role: "tool", tool_call_id, content}` messages.

### Sandbox (where supported)
- **MUST** implement the SandboxState machine (`NotInstalled → Downloading → Extracting → Installing → Ready / Error`).
- **MUST** sequence state transitions through a single mutator.
- **MUST** validate path traversal on every sandbox-relative file operation.
- **MUST** expose `cancel()` that aborts in-flight setup.
- **MUST** surface tar-extraction errors (including symlink failures) in the post-extract state.
- **SHOULD** retry mirror downloads with single-retry-on-IOException and exponential backoff between user-driven retries.

### Daemon (where supported)
- **MUST** auto-start when the daemon-enabled flag is on (or be a no-op if the platform forbids long-running background work).
- **MUST** be idempotent on re-assert.
- **MUST** expose a "daemon-paused" state to the UI when the platform forces a stop.
- **MUST** log exception types from foreground-service-start failures, not swallow them under generic `Exception`.

### Heartbeat
- **MUST** dedup pending heartbeat notifications (replace-on-new with a fixed id).
- **MUST** offer **per-app gating** for the LLM-forward path of notification content. Default to **OFF for cloud LLMs**, **ON for on-device LLMs only**.
- **MAY** offer secret-pattern redaction (6-digit OTP near the word "code" → redact) as a defence-in-depth.

### Security baseline (closes port-CF1)
- **MUST** default `cleartextTrafficPermitted=false` (or platform equivalent). HTTP support is per-host opt-in via a settings flag; HTTPS is the default for every provider URL and every MCP server URL.
- **MUST** store secrets (LLM API keys, email passwords, encryption keys) in the platform's secure enclave (Keychain / DPAPI / libsecret / Keystore). **NEVER** as plaintext files.
- **MUST** offer a "sanitised export" mode that omits secrets; the import flow re-prompts for them.
- **MUST** require explicit per-app opt-in for forwarding system notifications to LLM providers; the default is "no apps."
- **MUST** add `_schema_version: 1` to every export and refuse newer-than-current imports with a clear error.

### Settings
- **MUST** parse settings JSON strictly (no silent unknown-key drop). Unknown keys surface a user-visible warning.
- **MUST** allow concurrent reads/writes safely.
- **MUST** make migrations transactional (no partial state on failure).

### UI
- **MUST NOT** synchronously block the calling thread on resource-string lookup inside callback paths. Resource strings are pre-fetched at app start.

---

## Protocols and Persisted State

Wire shapes are pinned in `findings/protocols/protocols-and-state.md` §B1–B5. Reimplementation MUST preserve—

- The **proot argv shape** (12 elements, see protocols §B1) for any port that uses the same proot binary. A port using a different sandbox primitive re-derives this entirely.
- The **OpenAI-compatible request/response DTOs** (see protocols §B2a) for cross-provider robustness.
- The **MCP JSON-RPC 2.0 transport** (see protocols §B3).
- The **sandbox state machine** (see protocols §SM-1).
- The **daemon FGS lifecycle state machine** (see protocols §SM-2) on platforms that have an FGS-equivalent.
- The **heartbeat notification dedup** (one pending at a time, replace-on-new, see protocols §SM-3).

Persisted state must include `_schema_version` at every storage boundary that is exported (settings export today; conversation-store and memory-store should adopt the same convention going forward — *known unknown* `port-OQ2`).

---

## External Dependencies

| Dependency | Stance | Rationale |
|---|---|---|
| LLM HTTP libraries (OpenAI / Anthropic / Gemini / generic OpenAI-compat) | replace | Use the target stack's idiomatic HTTP client. |
| MCP transport | replace | Standard JSON-RPC 2.0; trivial to reimplement. |
| Sandbox primitive | re-design | The Kai 9000 source uses **Android proot+ptrace**. Port should choose per target— keep proot on Android, use Linux-namespaces on Linux desktop, Lima/Docker on macOS, WSL on Windows, or **declare sandbox a non-goal on platforms without an obvious primitive** (this is what Kai 9000 does for iOS/web). Closes `port-CF2`. |
| Encrypted key-value store | wrap | Each platform has a native— Keystore on Android, Keychain on macOS, DPAPI on Windows, libsecret on Linux. NEVER roll your own. |
| Compose Multiplatform UI | replace or postpone | The shape is portable but a port to Swift+TypeScript+native would re-implement the views per platform. |
| FileKit (file picker) | replace | Each platform has a native picker; FileKit is a convenience wrapper. |
| termux/proot | wrap (only if Android sandbox is in scope) | Pin a specific commit; rebuild from source for F-Droid reproducibility. |
| LiteRT / TensorFlow Lite (on-device LLM) | postpone or replace | Use the target stack's on-device-inference solution; on-device is **optional**, the port is functional with cloud-only providers. |
| Compose Multiplatform Resources | replace | Port to the target stack's i18n resource system. Pre-fetch strings at app start to avoid the `runBlocking` hazard. |
| Koin DI | replace | Use the target stack's idiomatic DI. |
| Ktor | replace | Use the target stack's idiomatic HTTP client. |

---

## Portability Hazards

(Consolidated in `findings/porting/reverse-engineering-bundle.md` §Portability Hazards. The 17 hazards listed there are the canonical reference; not duplicated here.)

The two highest-impact decisions are—
1. **Sandbox primitive** (port-CF2) — see §External Dependencies above. Each platform picks one or declares non-goal.
2. **Background execution model** — Android FGS does not transfer. iOS uses `BGTaskScheduler` / `BGAppRefreshTask`; desktop can use a native daemon, system-tray-hosted process, or `cron`/`launchd`/`systemd` registration; web only runs heartbeat while the page is open (or via a service worker with limited capabilities).

---

## Implementation Sequence

### Module Order

1. `settings-store` (with strict JSON + secure-enclave key storage) — every other module reads from it.
2. `chat-storage` and `memory-store` (encrypted at rest via `settings-store`'s key handle).
3. `provider-chain` (OpenAI-compatible first; Anthropic and Gemini incrementally).
4. `tool-registry` (built-in tools— web-search, calendar, send-notification).
5. `mcp-client` (JSON-RPC 2.0; gates the MCP-tools entry into `tool-registry`).
6. `agent-core` (chat orchestration; depends on all the above).
7. `ui-shell` (chat + settings screens; depends on agent-core).
8. `heartbeat-scheduler` (depends on agent-core + memory-store).
9. `notification-store` (per-platform listener; only where supported).
10. `daemon` (per-platform foreground/background-execution; only where supported).
11. `settings-export` (depends on settings-store + chat-storage + memory-store).
12. `sandbox` (depends on chat-storage for transcript persistence; per-platform sandbox primitive — only where supported).

### Scope Tiers

#### v0 (minimum viable port)
- Modules 1–7 from the list above (settings-store, chat-storage, memory-store, provider-chain, tool-registry, agent-core, ui-shell).
- Built-in tools— chat only (no sandbox, no notification listener, no MCP).
- Single LLM provider (the user's choice; no fallback chain in v0).
- Settings export with `_schema_version` and **sanitised-only** mode.
- Default `cleartextTrafficPermitted=false`. Platform secure-enclave key storage.
- **All four HIGH-security findings (port-CF1) are addressed in v0** — they are baseline requirements, not v1 deferrals.
- No daemon. No heartbeat. No notification listener.

#### v1 (major-workflow parity)
- Add provider-chain fallback (24+ providers).
- Add `mcp-client` and MCP tools in `tool-registry`.
- Add `heartbeat-scheduler` (cloud-LLM heartbeat **only** when the user has explicitly opted in — per-app allowlist for notification content; defect 4.4 design baked in).
- Add `daemon` on platforms with a long-lived background surface (Android FGS; iOS BGTaskScheduler; desktop system-tray).
- Add `settings-export` "full" mode (with the cleartext-warning UI).
- Optional— on-device LLM provider (LiteRT-equivalent).

#### v2 (full parity with Kai 9000)
- Add `sandbox` on the platform(s) the port chooses (Android by default; per-platform primitive otherwise).
- Add `notification-store` on platforms that allow system notification access.
- Add advanced tool features (image attachments, calendar event creation, settings export round-trip with conversations).

---

## Acceptance Scenarios

Black-box checks. Independent of stack choice.

| # | Scenario | Input | Expected Output / Side Effect |
|---|----------|-------|-------------------------------|
| 1 | Send chat message via single configured provider | API key configured for one provider; user types "Hello"; presses send | Streamed assistant response appears; conversation transcript persisted encrypted. |
| 2 | Provider transport failure → fallback | Provider 1 (configured) unreachable; provider 2 configured; user sends message | Provider 2 receives the request; assistant response appears. |
| 3 | Tool call dispatch | LLM returns `tool_calls=[{name:"web_search", arguments: "{\"q\":\"x\"}"}]` | `tool-registry` dispatches to web-search; result returned as `{role:"tool", tool_call_id, content}` next message. |
| 4 | Tool call with parsed-JSON arguments | LLM returns arguments as already-parsed JSON object instead of string | Accepted without error (defect 5.1 fix). |
| 5 | Settings export with `_schema_version` | User taps Export; chooses sanitised mode | JSON file produced with `_schema_version: 1` at top; `api_key` and `email_passwords` fields are absent or set to a placeholder. |
| 6 | Settings import refuses newer schema | User imports a file with `_schema_version: 2` into a v1-compatible build | Import is refused with a "this export was created by a newer version" error. |
| 7 | Settings import strict on unknown keys | Import file contains a section the running build doesn't know | User-visible warning— "Some sections were unrecognised— <list>"; recognised sections are still imported. |
| 8 | Cleartext HTTP refused by default | User configures provider with URL `http://example.com/...` | Connection refused with an error— "Use HTTPS or enable cleartext for example.com in settings". |
| 9 | Cleartext HTTP per-host opt-in | User enables cleartext for `127.0.0.1` and configures Ollama at `http://127.0.0.1:11434` | Connection succeeds. |
| 10 | Encryption key stored in secure enclave | Inspect on-disk store on each platform | No file contains the AES-256 key; the key is in Keychain / DPAPI / libsecret / Keystore. |
| 11 | Sandbox setup happy path (where supported) | User taps Setup; network available | State progresses Downloading → Extracting → Installing → Ready; Terminal becomes interactive. |
| 12 | Sandbox setup all mirrors fail | Network unreachable to all 6 mirrors | State reaches Error; partial download deleted; retry button visible. |
| 13 | Sandbox path traversal blocked | Tool tries `executeCommand` with `..//etc/passwd` arg | Path resolution returns null; no host-OS path touched. |
| 14 | Sandbox state mutation under concurrent triggers | User taps Cancel while installPackages() is in flight | State transitions are serialised; no torn or unintended state. |
| 15 | Daemon survives backgrounding (where supported) | User backgrounds the app | Persistent notification visible; heartbeat fires per schedule. |
| 16 | Daemon time-limit recovery (where supported) | Platform forces FGS stop | "Daemon paused" UI signal posted; if the activity is foregrounded, daemon re-asserts immediately (defect 5.5 fix). |
| 17 | Heartbeat with on-device LLM only | User has only an on-device LLM in chain; system notification arrives matching capture filter | Notification content used for heartbeat context. |
| 18 | Heartbeat with cloud LLM and no per-app allowlist | User has cloud LLM in chain; no per-app allowlist for forwarding | Notification content **NOT** sent to cloud LLM. |
| 19 | Heartbeat with cloud LLM and per-app allowlist | User explicitly allowlists "Slack" for LLM forward; Slack notification arrives | Notification content sent to cloud LLM as heartbeat context. |
| 20 | Settings export + reimport round-trip | Configure settings; export sanitised; wipe install; reinstall; import + re-enter API keys | Provider chain, soul, memory, MCP servers, etc. restored; secrets re-entered manually. |
| 21 | Heartbeat dedup is idempotent-by-design | Configuration change while heartbeat notification is pending; re-trigger via deep-link | The deep-link is consumed exactly once (defect 5.2 fix); no double-load. |
| 22 | iOS / desktop / web sandbox stub (where sandbox is non-goal) | Any non-supported platform | Sandbox features are not exposed in the UI (or are exposed with "not supported on this platform" messaging). |

---

## Deliberate Non-Goals

- **Sandbox on iOS / web** (per `port-CF2` decision)— the reimplementation may keep the Kai 9000 asymmetry (Android-only sandbox) OR may add a sandbox to a desktop platform, but **iOS and web ship with no sandbox**. iOS forbids `ptrace` and arbitrary native execution; web has no equivalent primitive.
- **Conversation export inside `settings-export`'s sanitised mode**— conversations are user content; the sanitised export omits them by default.
- **No top-level user account / cross-device sync**— Kai is local-first.
- **No analytics / telemetry**— explicitly absent from the source.
- **No retroactive heartbeat replay** when daemon is killed— missed heartbeats stay missed.
- **No automatic battery-whitelist request**— matches Kai 9000 design (rely on FGS + re-assertion).
- **No persistent SAF / equivalent file-bookmark**— file pickers are one-shot reads only.
- **No automatic cross-version migration in v0**— the import-refuse-newer-schema rule pushes upgrade burden onto the user (acceptable tradeoff for v0).

---

## Known Unknowns

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| spec-OQ1 | needs-runtime-test | Whether on-device LLM (LiteRT-equivalent) inference latency is acceptable for heartbeat-frequency calls. | Spike before v1 commits to on-device fallback. |
| spec-OQ2 | needs-spec-ruling | Conversation-record on-disk format details (record framing, IV/nonce placement, key wrap). | Inherited from `port-OQ2`. Spike before chat-storage v0. |
| spec-OQ3 | needs-runtime-test | Whether mirror-list user-extension (defect 6.1 fix) introduces malicious-mirror risk; mitigation = explicit warning UI. | UX spike. |
| spec-OQ4 | needs-maintainer-decision | Whether v2 ships notification listener at all (the policy / privacy story is heavyweight; defect 4.4 baseline already constrains it). | Decide at v1 close. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| spec-CF1 | spike | On-device LLM inference latency benchmark. | Need running spike before v1. |
| spec-CF2 | spike | Conversation-record on-disk format design. | Need running spike before chat-storage v0. |

## Spike List

1. **Encrypted-store-per-platform spike.** Verify Keychain (macOS) / DPAPI (Windows) / libsecret (Linux) / Keystore (Android) integration; measure key-derivation overhead on cold-start. Output → `chat-storage` v0 viability.
2. **Sandbox primitive spike (Android).** Verify proot binary still cross-compiles cleanly with the chosen NDK version; measure rootfs extract time on a low-end device. Output → `sandbox` v2 schedule.
3. **Provider-chain provider drift spike.** Run the OpenAI-compatible adapter against 3 randomly-chosen providers from Kai's list (Ollama, OpenRouter, Together) and a fourth that's known to return parsed-JSON arguments (Gemini). Output → confirms `function.arguments` accept-both-shapes design.
4. **Heartbeat-with-on-device-LLM spike.** Ensure the constraint "do NOT forward notification content to cloud LLMs without per-app opt-in" is enforceable cleanly in the heartbeat-scheduler module. Output → `heartbeat-scheduler` v1 design.
5. **Settings-import strict-parse UX spike.** Build the warning UI for unknown keys; tune wording so users don't panic on benign forward-compat warnings.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | Concept-level modules are defined. | PASS | §Conceptual Module Model defines 12 modules with the required fields (responsibility, public inputs, public outputs, owned state, invariants, collaborators). |
| 2 | Required behaviors are stated. | PASS | §Required Behaviors covers chat / tools / sandbox / daemon / heartbeat / **security baseline** / settings / UI with explicit MUST / SHOULD / MAY pinning. |
| 3 | Protocol and persisted state expectations are stated. | PASS | §Protocols and Persisted State references the canonical protocols phase output and pins the must-preserve elements. |
| 4 | Acceptance scenarios and known unknowns are included. | PASS | §Acceptance Scenarios has 22 numbered scenarios; §Known Unknowns has 4 entries with deferred reasons. |
| 5 | Defects identified in either scan are explicitly designed-around or noted as "left behind", with the choice cited. | PASS | Every "fix-before-porting" defect from the porting bundle is referenced in §Required Behaviors → relevant subsection (e.g., defect 4.1 fix → "MUST default `cleartextTrafficPermitted=false`"; defect 4.4 → heartbeat per-app gating MUST; defect 5.3 → `_schema_version` MUST). "Port-differently" defects are referenced in module invariants. "Leave-behind" defects appear in §Deliberate Non-Goals or are implicit in the spec's silence on them. |
| 6 | Findings are marked with evidence levels. | PASS | Every claim either cites the source phase output (architecture, contracts, protocols, mech-defects, semantic-defects, porting bundle) or is a normative MUST / SHOULD / MAY rule whose evidence trace lives in the cited phase output. |

**Validated by:** 2026-05-06 (reimplementation-spec, session 7)
**Overall:** PASS
