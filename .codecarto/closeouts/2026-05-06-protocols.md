# Closeout — 2026-05-06 — protocols

## Summary

Fourth implementing session. Drove the protocols phase— produced `findings/protocols/protocols-and-state.md` (433 lines). Closed all three contracts→protocols `carry_forward` items (ctr-CF1 proot argv, ctr-CF2 AI tool-call JSON, ctr-CF3 settings export JSON). Documented 8 boundaries, 4 in-depth event catalogs (proot, OpenAI-compatible LLM HTTP, MCP JSON-RPC, settings export), 3 state machines, and 10 compatibility hazards.

## Files Touched

- **Added:** `.codecarto/findings/protocols/protocols-and-state.md`; `.codecarto/closeouts/2026-05-06-protocols.md` (this file).
- **Modified:** `.codecarto/workflow/status.yaml` (protocols → complete; current_phase → defect-scan-semantic; proto-CF1/2/3 routed); `.codecarto/THREAD_LOG.md`.

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block on primary output | PASS | All 5 protocols criteria PASS. |
| Carry-forward closure | PASS | All 3 contracts→protocols entries explicitly closed. |
| Required reads loaded | PASS | architecture-map.md + behavioral-contracts.md + mechanical-defects.md all consumed. |

## Decisions Beyond Prompt

- **D-9** | Documented the OpenAI-compatible DTO shape in detail; deferred Anthropic / Gemini specifics to proto-OQ1. | The OpenAI-compatible shape carries the bulk of the 24 providers; the others are out-of-scope for the wire-format extraction this phase needs to do. Porting can pick those up later.
- **D-10** | Surfaced "no top-level `version` field in settings export" as a flagged finding routed to porting via proto-CF3, rather than just an open question. | This is an architectural decision the port should consciously override; flagging it forward via carry_forward keeps the porting phase aware.

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| proto-CF1 | defect-scan-semantic | Settings-import key-drop hazard (Pass 5). |
| proto-CF2 | defect-scan-semantic | function.arguments JSON-string drift across providers (Pass 5). |
| proto-CF3 | porting | Add `_schema_version` to the export format. |

## Next Session Pointer

Defect-scan-semantic next (`current_phase: defect-scan-semantic`). Required reads— architecture-map.md + behavioral-contracts.md + protocols-and-state.md + mechanical-defects.md. Pick up— mech-CF1, mech-CF2, mech-CF3, proto-CF1, proto-CF2 (all targeting this phase).
