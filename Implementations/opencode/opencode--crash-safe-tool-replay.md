---
type: implementation
harness: opencode
concept: crash-safe-tool-replay
commit: ecc4916b5a
files: [packages/core/src/session/runner/llm.ts:119-139, packages/core/src/session/runner/llm.ts:395-399, specs/v2/session.md:50, specs/v2/todo.md:56-74, packages/opencode/src/session/processor.ts:595-606, packages/opencode/src/session/message-v2.ts:362-373]
---
[[crash-safe-tool-replay]] in [[opencode]].

## Mechanism

### Legacy runtime
- No durable intent record. An aborted run marks in-flight tool parts `status: "error"`, `error: "Tool execution aborted"`, `metadata.interrupted: true` (`packages/opencode/src/session/processor.ts:595-606`).
- A tool part still `pending`/`running` at replay (process died mid-call) is sent as `output-error` "[Tool execution was interrupted]" so every `tool_use` has a result (`packages/opencode/src/session/message-v2.ts:362-373`). Nothing is rerun.

### v2 runtime
- Tool calls are durably projected (`Tool.Called`) before the tool fiber starts (`specs/v2/session.md:50`).
- Before each drain, `failInterruptedTools` publishes `SessionEvent.Tool.Failed` with `"Tool execution interrupted"` for every tool still `pending`/`running` from a previous process (`packages/core/src/session/runner/llm.ts:119-139`, called at `:399`). "Abandoned side effects are never silently replayed" (`specs/v2/session.md:50`).
- No per-tool replay policy. Retry/abandon decisions and "bounded automatic retry only where provider and tool idempotency make it safe" are a design TODO (`specs/v2/todo.md:56-74`).

## Constants
| name | value | path:line |
|---|---|---|
| v2 interrupted error text | `Tool execution interrupted` | `packages/core/src/session/runner/llm.ts:131` |
| legacy abort error text | `Tool execution aborted` | `packages/opencode/src/session/processor.ts:602` |
| legacy replay text | `[Tool execution was interrupted]` | `packages/opencode/src/session/message-v2.ts:370` |

## Evolution
- 2026-04-09 `c29392d085` legacy keeps interrupted bash partial output in the replayed result.
- 2026-05-25 `748fcb7ebd` cleanup-marked interrupted tools no longer count as pending for loop continuation.
- 2026-06-03 `76ee87ead8` v2 runtime lands with `failInterruptedTools`.

## Quirks / drift
- Fail-all: safe, idempotent tools also lose their result after a crash; the model must re-issue them.
- The spec explicitly refuses an enclosing durable execution identity (`specs/v2/todo.md:73-74`).

Failures: [[non-idempotent-tool-replayed-after-crash]] · [[orphaned-tool-calls-and-results]].

Contrast: [[pi--crash-safe-tool-replay|pi]] durable reruns tools whose stored and current policy are both `safe`; opencode v2 never reruns, it only records the interruption.
