# Closeout — 2026-05-06 — architecture

## Summary

First implementing session for the Kai project. Drove the architecture phase of `workflow/pipeline.yaml` (5-phase, no defect scan) and produced `findings/architecture/architecture-map.md`. Mapped the Kotlin Multiplatform / Compose Multiplatform layout (`:composeApp` shared core + `:androidApp` shell + `iosApp/` Xcode shell + `:screenshotTests`) and the four user-named focus subsystems pre-seeded as `carry_forward` in `status.yaml`: Alpine/proot bootstrap, Android foreground daemon, SAF binds (delegated to FileKit), and battery whitelist (deliberate non-request, mitigated at the FGS layer). Two surprise findings worth flagging: SAF integration is **delegated to the FileKit library** rather than implemented directly, and the project **never asks for `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`** — it relies on `START_STICKY` + `MainActivity.onStart()` re-assertion. Next session is the contracts phase, which inherits the four focus carry_forward entries.

## Files Touched

- **Added:**
  - `.codecarto/findings/architecture/architecture-map.md` (primary output, 380 lines)
  - `.codecarto/closeouts/2026-05-06-architecture.md` (this file)
- **Modified:**
  - `.codecarto/workflow/status.yaml` (architecture → complete, current_phase → contracts, owner_notes added)
  - `.codecarto/THREAD_LOG.md` (one-line index entry appended)
- **Deleted:** none.

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block on primary output | PASS | All 5 criteria PASS; no PARTIAL/FAIL rows. |
| All required reads loaded | PASS | GUIDE.md, status.yaml, pipeline.yaml, architecture SKILL.md, architecture template all consumed before writing. |
| Focus subsystem coverage | PASS | All 4 carry_forward subsystems have a clearly-labelled subsection in §Focus subsystem deep-dives. |
| Source-code-as-evidence discipline | PASS | Every claim tagged *observed fact* / *strong inference* / *portability hazard*; two genuinely open questions captured in §Open Questions. |

## Decisions Beyond Prompt

- **D-1** | Switched the active pipeline from the framework default `pipeline-full-with-deep-audit.yaml` to the 5-phase `pipeline.yaml` (no defect scan). | The user's stated goal is "produce architecture maps and behavioral contracts in this run, with focus on Alpine/proot, foreground service, SAF binds, battery whitelist." Running a defect-scan-mechanical phase between architecture and contracts would lengthen the session without serving that goal. Documented in the framework drop-in commit message.
- **D-2** | Pre-seeded the four focus subsystems as `carry_forward` entries in the architecture phase's slot of `status.yaml` rather than open-coding them in the architecture-map. | Status.yaml is the canonical handoff baton; the contracts phase reads it on session start. Subsystem subsections in the architecture map mirror the carry_forward IDs (`kai-CF-alpine-proot`, etc.) for traceability.
- **D-3** | Kept all four subsystem sections in the primary `architecture-map.md` rather than splitting them into the secondary `findings/runtime-lifecycle/`, `findings/state-and-storage/`, etc. files. | The primary output is the load-bearing reference for the contracts phase; secondary append-mode files are for catalog-level detail. The four focus subsystems are user-flagged priorities and belong in the primary output.

## Proposed Conventions

### C? — "User-named focus areas seed status.yaml carry_forward up front"

**Why:** When a user identifies named focus subsystems before the run, capturing them as explicit `carry_forward` entries on the *first* phase (architecture) ensures every later phase loads them on session start (per GUIDE.md "Session Update Protocol" step 7), rather than relying on the architecture map alone to surface them.

**How to apply:** During the framework drop-in / session-zero commit, write the focus areas into `status.yaml` under the architecture phase's `carry_forward:` block as `defer-to-phase` entries with explicit `target_phase` values. The architecture map then mirrors them in dedicated subsections.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ1 | needs-runtime-test | `LinuxSandboxManager.makeWritable()` symlink behavior. | See architecture-map.md §Open Questions. |
| arch-OQ2 | needs-maintainer-decision | Is `:screenshotTests` CI-only or also locally runnable? | See architecture-map.md §Open Questions. |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| kai-CF-alpine-proot | contracts | Alpine/proot user-visible setup contract. | See architecture-map.md §Focus / Alpine. |
| kai-CF-foreground-service | contracts | Daemon notification visibility, OS-kill recovery contract. | See architecture-map.md §Focus / Foreground service. |
| kai-CF-saf-binds | contracts | SAF permission flow (delegated to FileKit) + FileProvider outbound contract. | See architecture-map.md §Focus / SAF binds. |
| kai-CF-battery-whitelist | contracts | Behavior under Doze and OEM aggressive battery management with no whitelist request. | See architecture-map.md §Focus / Battery whitelist. |

## Next Session Pointer

The contracts phase runs next (`current_phase: contracts` in `workflow/status.yaml`). It should:

1. Read `findings/architecture/architecture-map.md` (especially §Focus subsystem deep-dives).
2. Read the four `carry_forward` entries this phase routed to it (`kai-CF-alpine-proot`, `kai-CF-foreground-service`, `kai-CF-saf-binds`, `kai-CF-battery-whitelist`) — each must become a clearly-labelled subsection of `behavioral-contracts.md`.
3. Use `templates/behavioral-contracts.md`'s feature-contract table shape (trigger / defaults / observable output / side effects / persisted state / error behavior / recovery behavior) for each focus subsystem feature.
4. Document Kai's security/auth model (LLM API key handling, encrypted local storage, notification-listener trust boundary) — relevant per the contracts pipeline's completion criteria 3.
5. Include a black-box acceptance list covering at minimum: a heartbeat round-trip after the app is force-stopped + reopened; a sandbox-Ready transition from a clean install; a sandbox-produced file opened via FileProvider; an FGS restart after `onTimeout`.
