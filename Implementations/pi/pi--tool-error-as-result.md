---
type: implementation
harness: pi
concept: tool-error-as-result
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:912, packages/agent/src/agent-loop.ts:717, packages/agent/src/agent-loop.ts:844, packages/agent/src/agent-loop.ts:898, packages/agent/src/types.ts:479, packages/coding-agent/src/core/tools/bash.ts:400, packages/coding-agent/src/core/agent-session.ts:667, packages/coding-agent/src/extensions/mcp/tools.ts:124, packages/durable/src/harness/tool.ts:93, packages/durable/src/tools/env.ts:4]
---
[[tool-error-as-result]] in [[pi]].

## Mechanism
- **Tool contract**: "Execute the tool call. Throw on failure, or return a result with `isError: true`; do not only describe the failure in `content`." (`packages/agent/src/types.ts:479-482`).
- `createErrorToolResult(message)` = `{content:[{type:"text", text: message}], details:{}}` paired with `isError:true` (`packages/agent/src/agent-loop.ts:912-917`). Every failure path in the pipeline funnels into it:

| failure | model-visible text | path:line |
|---|---|---|
| unknown tool | `Tool X not found` | `agent-loop.ts:717-723` |
| prepare/validation throw | the error message (validator: `Validation failed for tool "X": … Received arguments: …`) | `agent-loop.ts:770-775`; `packages/ai/src/utils/validation.ts:340-348` |
| `beforeToolCall` block | `reason` or "Tool execution was blocked" (+ optional `terminate`) | `agent-loop.ts:745-755` (`1eb988cfe` #7715) |
| extension `tool_call` handler throws | fail-closed block: "Extension failed, blocking execution: …" | `packages/coding-agent/src/core/agent-session.ts:667-679` |
| abort before/after preflight | "Operation aborted" | `agent-loop.ts:741,760`; parallel thunk guard `:623` (`afda4d620` #8935) |
| `execute` throws | `error.message` | `agent-loop.ts:844-852` |
| `afterToolCall` throws | `error.message` | `agent-loop.ts:898-901` (`b9cd557d1` #3084) |
| length-truncated assistant msg | "Tool call X was not executed: the response hit the output token limit…" | `agent-loop.ts:471-500` → [[truncated-tool-call-guard]] |

- `runToolCall` "never rejects for tool failures" (`agent-loop.ts:802-810`) → one tool's failure never aborts a parallel batch ([[parallel-tool-execution]]).
- **Return-isError (not throw) for results that must carry data**: bash non-zero exit returns `output + "\n\nCommand exited with code N"` + `structuredContent` + `isError:true` (`packages/coding-agent/src/core/tools/bash.ts:400-407`) so codemode scripts still get the structured value; MCP `isError` → error result for model but untruncated `structuredContent` for scripts; empty isError → "MCP tool s/t returned an error" (`packages/coding-agent/src/extensions/mcp/tools.ts:124-230`). Introduced with `8562bcf66` (2026-09-29).
- Tool-side errors are written as recovery instructions: read `Offset N is beyond end of file (M lines total)` (`read.ts:163-165`); edit errno classification `Could not edit file: {path}. Error code: ENOENT.` (`edit.ts:174-182`, `43ee9b77e`/`ebdf3cf45` #3894); grep `ripgrep (rg) is not available and could not be downloaded` (`grep.ts:119-123`); bash `Command timed out after N seconds` / `Command aborted` with partial output prepended (`bash.ts:374-380`).
- Spawn errors must not escape: bash checks cwd exists, attaches `child.on("error")` → reject (`1432fd91d` #479; before: uncaught ENOENT crashed the whole session).
- Null content normalized at ingestion: `content: result.content ?? []` (`agent-loop.ts:935-937`; `8c0ccd14b` #6343).
- Replay-side complement: orphaned tool calls get synthetic `"No result provided"` isError results at the provider boundary → [[transcript-replay-repair]].

## Constants
| name | value | path:line |
|---|---|---|
| block default reason | "Tool execution was blocked" | `packages/agent/src/agent-loop.ts:746` |
| abort text | "Operation aborted" | `packages/agent/src/agent-loop.ts:741` |

## Evolution
- 2025-11-11 `159075cad` "tools now throw exceptions instead of returning error messages".
- 2025-11-12 `f147109da` "Fix coding agent tools to return error content instead of throwing" → same day `9e3e319f1` back to `reject(new Error(...))`. Final design: tools throw, loop renders.
- 2026-01-05 `1432fd91d` (#479, #1230) bash spawn errors contained.
- 2026-03-14 `63ac2df24` (#2113) interception moved into agent-core `beforeToolCall/afterToolCall`.
- 2026-04-16 `b9cd557d1` (#3084) afterToolCall throw → error result instead of aborting batch.
- 2026-05-19 `b94482762` (#4276) abort re-checked after hooks.
- 2026-07-06 `8c0ccd14b` (#6343) null content normalization.
- 2026-08 `1eb988cfe` (#7715) `terminate` on block.
- 2026-09-29 `8562bcf66` isError *returns* for data-carrying failures (bash exit≠0, MCP).
- 2026-09 `afda4d620` (#8935) aborted guard in prepared parallel thunks.

## Evidence commits
`159075cad`, `f147109da`, `9e3e319f1`, `1432fd91d`, `63ac2df24`, `b9cd557d1`, `b94482762`, `8c0ccd14b`, `1eb988cfe`, `8562bcf66`, `afda4d620`, `43ee9b77e`, `ebdf3cf45`

## Quirks
- Error results have empty `details` → renderers/extensions get no structured error info unless the tool returned isError itself.
- Throw vs return asymmetry between harnesses: coding-agent bash *returns* isError on non-zero exit; durable bash *throws* "Command exited with code N" (error result still carries output) (`packages/durable/src/tools/bash.ts:109-116`).
- `tool_result` hooks see thrown errors too (0.31.0 fix in [[side-door-input-bypasses-hooks]]).

## Durable variant (packages/durable)
- `ExecutionEnv` returns `Result<T,E>` for expected failures, never throws (`packages/durable/src/env/index.ts:3-21`); `FileErrorCode` 8 values, `ExecutionErrorCode` + callback_error/shell_unavailable/spawn_error/timeout, error may carry `spillPath` (`:35-68`). Tools convert to error results; missing env → "No execution environment is configured" (`packages/durable/src/tools/env.ts:4-8`).
- Call to a tool the request did not offer → `tool_unavailable` result with no task (`spec.md:3576-3594`).
- Crash-interrupted unsafe tool → error result "Tool X was interrupted and may have partially run" (`packages/durable/src/harness/tool.ts:93-110`) → [[crash-safe-tool-replay]].

## Failures
- [[bash-spawn-errors-crash-session]]
- [[hook-throw-aborts-parallel-batch]]
- [[signal-killed-command-reported-success]]
- related (01): [[length-truncated-tool-calls-executed]]
