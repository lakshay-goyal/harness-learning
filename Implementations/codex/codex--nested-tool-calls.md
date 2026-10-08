---
type: implementation
harness: codex
concept: nested-tool-calls
commit: 622e9e3696
files: [codex-rs/core/src/tools/context.rs:56-66, e95abcdf49:codex-rs/core/src/tools/parallel.rs:180-218, codex-rs/tools/src/tool_executor.rs:51-78, codex-rs/code-mode-protocol/src/description.rs:23-47]
---
[[nested-tool-calls]] in [[codex]] — only code-mode scripts nest tool calls; they re-enter the **same router** as direct model calls.

## Mechanism
- Provenance: `ToolCallSource::{Direct, DirectPlaintextMessage, CodeMode{cell_id, runtime_tool_call_id}}` — "Runtime cell that issued the nested tool request"; the per-cell runtime id "is not the Codex tool call id because the runtime id only needs to be unique within one cell" (`codex-rs/core/src/tools/context.rs:56-66`). Trace source `call_trace::Source::CodeMode` (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:180-185`).
- Same pipeline: readiness wait, parallel RwLock gate, PreToolUse/PostToolUse hooks, approvals and sandbox apply to nested calls (`parallel.rs:196-218`) → [[parallel-tool-execution]], [[tool-call-gate]]; blocking PostToolUse hooks honored in code mode (`d7f298fe20` 2026-06-15) → [[tool-result-rewriting]].
- Reachability: `ToolExposure::is_available_in_code_mode` — Direct, Deferred, CodeModeOnly yes; DirectModelOnly / DeferredModelOnly / Hidden no (`codex-rs/tools/src/tool_executor.rs:51-95`). `request_user_input` and `new_context` are DirectModelOnly (a script can't ask the user or reset context).
- Results to the script: tool's `code_mode_result` (structured JSON when an output schema exists, e.g. `exec_command`, `clock.curr_time`) → [[structured-tool-output]]; only the script's own output items (`text()`, `image()`, `notify()`…) reach the model → [[code-mode]].
- No bounded per-parent record of nested calls attached to the `exec` result was found (pi has one) — unverified.

## Evolution
- 2026-03-09 `da616136cc` code mode (nested calls via `tools.*`); 2026-06-15 `d7f298fe20` PostToolUse blocking in code mode; 2026-07-30 `c126f206da` normalized-name collisions across dynamic/namespaced tools resolved.

## Versus pi
pi exposes `ctx.executeTool` to any tool, records ≤256 nested calls on the parent result and sums usage ([[pi--nested-tool-calls]]); codex nests only from code-mode cells, with cell-scoped provenance.
