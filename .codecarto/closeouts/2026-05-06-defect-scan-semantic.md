# Closeout — 2026-05-06 — defect-scan-semantic

## Summary

Fifth implementing session. Drove defect-scan-semantic— 18 findings (0 critical, 4 HIGH, 7 medium, 7 low). All five inherited `carry_forward` items closed (mech-CF1/2/3, proto-CF1/2). Validation PASSes all 6 semantic-pass criteria.

The four HIGH findings cluster in Pass 4 (security)— the most consequential are—
1. `cleartextTrafficPermitted="true"` globally in `network_security_config.xml`. The entire Android app permits HTTP.
2. Notification content from KaiNotificationListenerService is forwarded to whatever LLM provider the user has configured— if cloud, **2FA codes / payment confirmations / message previews leave the device**.
3. Desktop's `EncryptedFileSettings` stores the AES-256 key as plaintext bytes in `getAppFilesDirectory()/settings.key`— same-user malware decrypts everything.
4. Settings export includes API keys and email passwords in cleartext, no warning UI.

All 4 are flagged as **fix before porting**— they shouldn't survive the port. The Android encryption path (Jetpack Security `EncryptedSharedPreferences` + Keystore) is robust— only the desktop path has the key-file issue.

## Files Touched

- **Added:** `.codecarto/findings/defect-scan-semantic/semantic-defects.md`; `.codecarto/closeouts/2026-05-06-defect-scan-semantic.md` (this file).
- **Modified:** `.codecarto/workflow/status.yaml`— defect-scan-semantic → complete; current_phase → porting. `.codecarto/THREAD_LOG.md`.

## Decisions Beyond Prompt

- **D-11** | The desktop encryption key-file issue is flagged as separately scoped from the Android path. | The Android side (`Platform.android.kt:160-168`) uses Jetpack Security `EncryptedSharedPreferences` with `MasterKey.AES256_GCM` (hardware-bound). The defect is desktop-only and the fix is platform-Keychain-per-OS— this is a porting decision, not a single-target patch.
- **D-12** | Filter on KaiNotificationListenerService finding (4.4) was upgraded to HIGH severity due to the **outflow path to cloud LLM providers**, not just the local capture. | Local notification capture is permitted by the user (system-level grant); the leak vector is the cloud LLM pipeline, which the user cannot control per-notification.

## Carry-Forward Closed

| ID | From phase | Closed because |
|---|---|---|
| mech-CF1 | defect-scan-mechanical | 3.1 — runBlocking concurrency hazard. |
| mech-CF2 | defect-scan-mechanical | 5.2 — heartbeat-dedup idempotency contract. |
| mech-CF3 | defect-scan-mechanical | 3.2 — concurrent state writes in LinuxSandboxManager. |
| proto-CF1 | protocols | 4.5 — ignoreUnknownKeys silent-drop on settings import. |
| proto-CF2 | protocols | 5.1 — function.arguments cross-provider drift. |

## Next Session Pointer

Porting phase next. Required reads— architecture-map.md + behavioral-contracts.md + protocols-and-state.md + mechanical-defects.md + semantic-defects.md. Pick up proto-CF3 (no `_schema_version` in settings export— add to porting recommendations).
