---
type: implementation
harness: codex
concept: tool-error-as-result
commit: 622e9e3696
files: [codex-rs/tools/src/function_call_error.rs:3-10, e95abcdf49:codex-rs/core/src/tools/parallel.rs:106-110, e95abcdf49:codex-rs/core/src/tools/parallel.rs:332-361, codex-rs/core/src/tools/registry.rs:576-624, codex-rs/core/src/tools/registry.rs:888-893, codex-rs/core/src/stream_events_utils.rs:400-430, codex-rs/core/src/session/turn.rs:2480-2483, codex-rs/core/src/function_tool.rs]
---
[[tool-error-as-result]] in [[codex]].

## Mechanism
- Two error classes (`codex-rs/tools/src/function_call_error.rs:3-10`): `FunctionCallError::RespondToModel(String)` → becomes a tool output with `success: false` and the turn continues (`needs_follow_up = true`); `FunctionCallError::Fatal(String)` ("Fatal error: …") → `CodexErr::Fatal`, **ends the turn** with an error event (`codex-rs/core/src/stream_events_utils.rs:400-430`; `e95abcdf49:codex-rs/core/src/tools/parallel.rs:106`).
- `failure_response` keeps the call's **wire shape** (`parallel.rs:332-361`): tool_search → empty `ToolSearchOutput{status:"completed", execution:"client", tools:[]}`; custom tool → `CustomToolCallOutput{success:false}`; else `FunctionCallOutput{success:false}` → [[tool-wire-kinds]].
- Unknown tool → `unsupported_tool_call_message`: "unsupported call: {name}" / "unsupported custom tool call: {name}" as `RespondToModel` (`codex-rs/core/src/tools/registry.rs:576-594,888-893`). Bad JSON → "failed to parse function arguments: {err}" (e.g. `codex-rs/core/src/tools/handlers/mcp_resource.rs:378`).
- Payload/kind mismatch (handler doesn't accept the payload kind) → `Fatal` (`registry.rs:609-624`).
- PreToolUse hook block → `RespondToModel(message)` (model sees it as the output); PostToolUse block → "PostToolUse hook blocked the tool result" (`registry.rs:626-678,785-795`) → [[tool-call-gate]], [[tool-result-rewriting]].
- **Fatal is used deliberately for host-side failures** where a model retry is pointless: sleep failure, clock read, user-input channel / response serialization, plugin listing, task join errors ("tool task failed to receive") (`codex-rs/core/src/tools/handlers/sleep.rs:145`; `codex-rs/core/src/tools/handlers/request_user_input.rs:99`; `parallel.rs:332-334`).
- Cancellation maps to "tool call cancelled" (`codex-rs/core/src/function_tool.rs`); aborted calls get a synthesized paired output "aborted by user after Xs" so history keeps call/output pairing (`parallel.rs:363-381`) → [[abort-propagation]].
- Shell-specific: sandbox denial is a normal output (exit code + output); exec errors "exec_command failed: {err:?}" capped at 900 bytes (`codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:57,441-473`) → [[shell-execution]].
- In-flight tool future failure during drain → `error_or_panic("in-flight tool future failed during drain")`, not a turn failure (`codex-rs/core/src/session/turn.rs:2480-2483`).
- Missing outputs in persisted history (crash, cancel) are synthesized as "aborted" at prompt build (`codex-rs/core/src/context_manager/normalize.rs:21-115`; `1e3cad95c0`) → [[transcript-replay-repair]], [[crash-safe-tool-replay]].

## Constants
| name | value | path:line |
|---|---|---|
| exec error message cap | 900 bytes | `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:57` |
| Fatal prefix | "Fatal error: " | `codex-rs/tools/src/function_call_error.rs:8` |

## Evolution
- 2025-10-03 `33d3ecbccc` "refactor tool handling (#4510)" — registry + `FunctionCallError`.
- 2025-12-14 `1e3cad95c0` no panic on tool call without output (resume).
- 2026-04-13 `a5783f90c9` custom tool outputs kept in cleanup/retry on stream failure ([[orphaned-tool-calls-and-results]]).
- 2026-10-07 `18e28fe1b9` dynamic tool lifecycles completed on cancellation.

## Versus pi
pi: every failure is an `isError` result; a thrown tool is rendered as a result, the loop never dies on a tool ([[pi--tool-error-as-result]]). codex adds a **Fatal** class that ends the turn for host-side failures — a deliberate departure from "everything is a result".
