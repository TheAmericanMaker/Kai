# Closeout — 2026-05-06 — defect-scan-mechanical

## Summary

Third implementing session for the Kai project (first since switching back to the deep-audit pipeline). Drove `defect-scan-mechanical`— passes 1 (logic), 2 (error handling), and 6 (config) over Kai's source. Produced `findings/defect-scan-mechanical/mechanical-defects.md` (16 findings— 4 medium, 12 low; zero critical/high). Three items spotted with semantic flavor were routed to defect-scan-semantic via `carry_forward` (mech-CF1/2/3). Validation block PASSes all 6 mechanical-pass criteria.

Key takeaway— the codebase shows good defensive discipline (well-commented filesystem-walk catches, evidence-tagged design rationale in code comments, generally informative error logging). The medium findings cluster in two places— (a) the `runBlocking { getString(...) }` pattern used to fetch resource strings inside notification paths (HeartbeatNotifier, NotificationHelper init); (b) the three-source-of-truth sandbox limits.

## Files Touched

- **Added:** `.codecarto/findings/defect-scan-mechanical/mechanical-defects.md`; `.codecarto/closeouts/2026-05-06-defect-scan-mechanical.md` (this file).
- **Modified:** `.codecarto/workflow/status.yaml` (defect-scan-mechanical → complete; current_phase → protocols; mech-CF1/2/3 routed); `.codecarto/THREAD_LOG.md` (one-line entry appended).

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block on primary output | PASS | All 6 criteria PASS. |
| Carry-forward routing | PASS | 3 semantic-flavored items routed (mech-CF1/2/3). |
| Required reads loaded | PASS | architecture-map.md consumed. |

## Decisions Beyond Prompt

- **D-7** | Focused mechanical scan on androidMain + repo-root native-build script; spot-sampled commonMain. | The architecture map identifies androidMain as the thickest adapter; mechanical defects concentrate where platform-specific code lives. CommonMain UI/ViewModel layer is more declarative (Compose + StateFlow) and lower-yield for mechanical defect scanning.
- **D-8** | Recorded the `runBlocking { getString(...) }` pattern as a *mechanical* defect (Pass 2 / error handling) AND routed the concurrency dimension to defect-scan-semantic via mech-CF1. | The pattern is locally identifiable (mechanical), but the actual hazard depends on which dispatcher the caller is on (semantic). Splitting avoids double-counting; the semantic phase resolves mech-CF1 with the concurrency framing.

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| mech-CF1 | defect-scan-semantic | `runBlocking { getString(...) }` concurrency angle. |
| mech-CF2 | defect-scan-semantic | MainActivity Intent-mutation heartbeat-dedup robustness (Pass 5). |
| mech-CF3 | defect-scan-semantic | Concurrent `_state.value = ...` writes in LinuxSandboxManager (Pass 3). |

## Next Session Pointer

Protocols phase next (`current_phase: protocols`). Required reads— GUIDE.md, status.yaml, architecture-map.md, mechanical-defects.md. Pipeline.required_reads also says behavioral-contracts.md is needed, so include it. Pick up the three contracts→protocols `carry_forward` items (ctr-CF1/2/3)— they want wire-format detail for proot argv, AI tool-call JSON, and settings-export JSON.
