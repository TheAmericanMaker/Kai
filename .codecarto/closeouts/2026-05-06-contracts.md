# Closeout — 2026-05-06 — contracts

## Summary

Second implementing session for the Kai project. Drove the contracts phase of `workflow/pipeline.yaml` and produced `findings/contracts/behavioral-contracts.md` (319 lines). Pinned user-visible behavior across Kai's Compose UI surfaces (Chat, Settings, Settings export/import, Sandbox terminal) and dedicated each of the four focus subsystems a full contract entry with the standard 7+ fields (trigger / defaults / observable output / side effects / persisted state / error behavior / recovery behavior / owner). All four `carry_forward` entries from architecture are now closed; three new `carry_forward` items routed to the protocols phase for wire-format detail (proot argv shape, AI tool-call JSON schema, settings export JSON schema). Validation block PASSes all 6 contracts-phase criteria.

## Files Touched

- **Added:**
  - `.codecarto/findings/contracts/behavioral-contracts.md` (primary output, 319 lines)
  - `.codecarto/closeouts/2026-05-06-contracts.md` (this file)
- **Modified:**
  - `.codecarto/workflow/status.yaml` (contracts → complete, current_phase → protocols, owner_notes added; carry_forward closed for the 4 architecture-routed items; 3 new carry_forward items for protocols)
  - `.codecarto/THREAD_LOG.md` (one-line index entry appended)
- **Deleted:** none.

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block on primary output | PASS | All 6 criteria PASS; no PARTIAL/FAIL rows. |
| All required reads loaded | PASS | GUIDE.md, status.yaml, pipeline.yaml, contracts SKILL.md, contracts template, AND `findings/architecture/architecture-map.md` (per pipeline.yaml `required_reads`) all consumed before writing. |
| Carry-forward closure | PASS | All 4 architecture-routed carry_forward entries (kai-CF-alpine-proot / -foreground-service / -saf-binds / -battery-whitelist) explicitly closed in §Carry-Forward Closure section. |
| Focus subsystem coverage | PASS | All 4 focus subsystems have a full feature-contract table with the standard fields under §Focus subsystems. |
| Black-box acceptance discipline | PASS | 14 numbered scenarios, each with precondition / action / expected outcome; covers happy path + error path + the cross-platform stub behavior. |

## Decisions Beyond Prompt

- **D-4** | Used the contracts pipeline's 6-criteria validation rubric (the default for `pipeline.yaml`) rather than the lite variant's 5-criteria rubric. | Active pipeline is `workflow/pipeline.yaml` not `workflow/pipeline-lite.yaml`; the security-and-authorization criterion (#3) applies. The §Security and Authorization section pins LLM provider key handling, encrypted local storage, MCP trust boundary, notification-listener trust boundary, and the sandbox trust boundary explicitly.
- **D-5** | Treated the deliberate non-features (no persistent SAF grants; no battery whitelist request) as **first-class contracts** with their own contract tables rather than as gaps. | The user's framing pre-flagged these as focus subsystems; the absence is the contract. Future implementations need to design *for* the absence (or knowingly diverge), not assume Kai requests these things.
- **D-6** | Routed wire-format detail (proot argv, AI tool-call JSON, settings export JSON) to the protocols phase via three new `carry_forward` entries (`ctr-CF1` / `ctr-CF2` / `ctr-CF3`) rather than extracting it here. | Per the contracts SKILL.md guidance, "If behavior is hard to infer from docs alone, read the companion protocol skill next." The wire-format detail belongs in protocols.

## Proposed Conventions

### C? — "Deliberate non-features get a full contract entry, not a gap note"

**Why:** When a project explicitly chooses *not* to implement a thing the user might expect (here: `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`, persistent SAF grants), that absence is itself a contract. Implementations and users need to know what behavior to expect *given the absence*. A "gap" framing implies the non-feature should be filled in; a "contract" framing pins the deliberate decision and its observable consequences.

**How to apply:** When a focus subsystem turns out to be a deliberate non-feature, give it a full feature-contract table whose Trigger row reads "(Non-feature)" and whose Observable Output / Side Effects / Recovery rows describe what happens *because the feature is absent*. Tested in this pass on `kai-CF-saf-binds` (FileKit + FileProvider only; no persistable grants) and `kai-CF-battery-whitelist` (no permission request).

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| ctr-OQ1 | needs-runtime-test | Exact default values for daemon-enabled, heartbeat interval, default LLM provider. | Pinned in `AppSettings` defaults; not extracted in this phase. |
| ctr-OQ2 | needs-spec-ruling | Settings-import partial-parse failure UX. | Not extracted from source in this phase. |
| ctr-OQ3 | needs-runtime-test | Encrypted local storage — algorithm, key derivation, key location. | Not extracted from source in this phase; relevant to a future defect-scan-semantic. |
| ctr-OQ4 | needs-maintainer-decision | Whether the "no battery whitelist request" decision is documented user-facing (e.g., FAQ for aggressive-OEM users). | Not surfaced in README; possible UX gap. |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| ctr-CF1 | protocols | proot argv shape (`buildProcessArgs(...)`), env vars (`buildEnvVars(extraEnv)`), working-directory normalisation. | Wire-format extraction is the protocols phase's rubric. |
| ctr-CF2 | protocols | AI tool-call JSON schema (tool-call message, tool-result message, error shape, retry semantics). | Wire-format extraction is the protocols phase. |
| ctr-CF3 | protocols | Settings export JSON schema — field names, version field, migration policy. | Wire-format extraction is the protocols phase. |

## Carry-Forward Closed

| ID (was) | From phase | Closed because |
|---|---|---|
| kai-CF-alpine-proot | architecture | §Focus / Alpine pins the full setup contract (states, mirror order, all-fail error, no-backoff retry, cancellation, path-traversal protection). |
| kai-CF-foreground-service | architecture | §Focus / Foreground service pins notification visibility (low-importance ongoing), OS-kill recovery (`START_STICKY` + `MainActivity.onStart` re-assertion), Android 14+ FGS time-limit (`onTimeout` clean-stop), Android 12+ background-start exception (silent swallow). |
| kai-CF-saf-binds | architecture | §Focus / SAF binds pins the FileKit picker contract (one-shot read; no persistable URI), the FileProvider outbound-share contract (`Intent.ACTION_VIEW` + read-grant), and the deliberate absence of persistent SAF grants. |
| kai-CF-battery-whitelist | architecture | §Focus / Battery whitelist pins the deliberate non-request contract — Doze/Standby behavior unmodified, OEM-kill silent, recovery via re-assertion next foreground. |

## Next Session Pointer

The protocols phase runs next (`current_phase: protocols` in `workflow/status.yaml`). It should:

1. Read `findings/architecture/architecture-map.md` AND `findings/contracts/behavioral-contracts.md` (both required by pipeline.yaml).
2. Pick up the three `carry_forward` entries this phase routed: `ctr-CF1` (proot argv), `ctr-CF2` (AI tool-call JSON), `ctr-CF3` (settings export JSON).
3. Document the protocol surfaces this contracts phase identified — proot subprocess interface, AI tool-call/tool-result JSON, MCP transport, settings export format, encrypted-storage on-disk shape.
4. State machine: the sandbox state machine (`NotInstalled / Downloading / Extracting / Installing / Ready / Error`) is documented at the contract level here; the protocols phase should pin transition guards and side effects per `templates/protocols-and-state.md`.
5. Compatibility hazards: Android FGS-rules-by-OS-version, OEM battery-manager differences, FileKit's per-platform back-end differences are all documented at the contract level here; the protocols phase consolidates them into the §Compatibility Hazards table.
