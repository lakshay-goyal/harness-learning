---
type: implementation
harness: pi
concept: structured-tool-output
commit: b30a6dd77
files: [packages/agent/src/types.ts:424-446, packages/agent/src/types.ts:474-482, packages/agent/src/agent-loop.ts:688-690, packages/agent/src/agent-loop.ts:929-942, packages/ai/src/types.ts:600-623, packages/coding-agent/src/core/tools/bash.ts:52-62, packages/coding-agent/src/core/tools/read.ts:28-38, packages/coding-agent/src/extensions/codemode/execute.ts:400-410, packages/coding-agent/examples/extensions/structured-output.ts:1-48, packages/evals/evals/documentation-audit.eval.ts:9-30, packages/durable/src/harness/types.ts:129-153]
---
[[structured-tool-output]] in [[pi]].

## Mechanism
- **Result shape** `AgentToolResult` (`packages/agent/src/types.ts:424-446`):
  - `content: (Text|Image)[]` — model-facing.
  - `details: T` — "Arbitrary structured details for logs or UI rendering"; persisted (JSON-typed, compile-time checked via `IsJsonCompatible`, `packages/ai/src/types.ts:607`); never sent to the model. Used e.g. edit `{diff, patch, firstChangedLine}`, MCP `{server, tool, fullOutputPath}`, tool_search `{loaded}`, todo state ([[branch-scoped-extension-state]]).
  - `structuredContent?: JsonValue` — "Machine-readable result matching the tool's `outputSchema`, for programmatic callers. Not sent to the model; `content` remains the model-facing result."
  - `usage?` — tool's own usage (not context accounting); `isError?` — "Report a failure without throwing … `details` and `structuredContent` are kept for the UI and programmatic callers" (`8562bcf66`); `terminate?`.
- `outputSchema?: TSchema` on `AgentTool` — "Tools that declare it should always set `structuredContent`" (`types.ts:474-477`); passed through `wrapToolDefinition` (`packages/coding-agent/src/core/tools/tool-definition-wrapper.ts:8-30`).
- **Persisted message** `ToolResultMessage` (`packages/ai/src/types.ts:609-623`): `content, details, usage, nestedCalls, isError, timestamp, durationMs` — **no `structuredContent`** (`createToolResultMessage`, `packages/agent/src/agent-loop.ts:929-942`); `nestedCalls` "Kept for the session record; not sent to the model"; `durationMs` monotonic clock excluding hooks (`36a686ee8`).
- **Built-in structured values**:
  - bash `outputSchema` `{output, truncated, full_output_path?, exit_code, wall_time_seconds}`; "A non-zero exit code is an error result for the model, but scripts still resolve to this value. `output` is not limited like the model-facing output" — up to `STRUCTURED_OUTPUT_MAX_BYTES = 1 MiB`, first/last 512 KiB around an omission marker (`packages/coding-agent/src/core/tools/bash.ts:24,52-62`; `1ff5b6fdd`). Hence bash returns `isError:true` instead of throwing on non-zero exit (`bash.ts:400-407`, `8562bcf66`).
  - read `outputSchema` = `string | {type:"image", data, mimeType, note}` (`read.ts:28-38`; `021eae60a` #10251).
  - MCP: `structuredContent` = full `CallToolResult` minus `_meta`, untruncated, even when `isError` ([[pi--mcp-integration|pi mcp-integration]]).
- **Consumer: codemode** `toScriptValue` — tool with `outputSchema` resolves to its `structuredContent` (also for error results that carry one); others resolve to text; `isError` without structured → reject (`packages/coding-agent/src/extensions/codemode/execute.ts:400-410`). Codemode renders `outputSchema` as TS return types in its description (`codemode/tool.ts:327`) — [[code-mode]].
- **Hooks**: replacing `content` in `afterToolCall`/`tool_result` without returning `structuredContent` drops it (`agent-loop.ts:884-895`) — [[tool-result-rewriting]].
- **Terminating tool**: `terminate: true` = "Hint that the agent should stop after the current tool batch. Early termination only happens when every finalized tool result in the batch sets this to true" (`types.ts:441-445`; `shouldTerminateToolBatch` `agent-loop.ts:690-692`); runtime-only — persisted toolResult is a standard result; skips only the automatic follow-up request, steering/follow-up still polled (01-loop findings `agent-loop.ts:272,295-308`). `beforeToolCall` block may also set `terminate` (`1eb988cfe` #7715). → [[turn-loop]].
  - Example `structured_output` tool: "Demonstrates `terminate: true` so the agent can end on a tool call without paying for an extra follow-up LLM turn"; data in `details`, guideline "After calling structured_output, do not emit another assistant response in the same turn." (`packages/coding-agent/examples/extensions/structured-output.ts:1-48`).
  - Eval use: `submit_documentation_audit` — `verdict: match|mismatch|inconclusive`, `explanation` 1-2000 chars, `additionalProperties:false`, `constrainedSampling: {type:"json_schema", strict:"prefer"}`, returns `details: params`, `terminate: true`; harness tools `read, grep, find, ls, submit_documentation_audit` (`packages/evals/evals/documentation-audit.eval.ts:9-41`) — [[harness-evals]], [[constrained-tool-sampling]].

## Durable variant (packages/durable)
- `ToolControl {terminate?: true; handoff?: string}` on results (`packages/durable/src/harness/types.ts:129-153`): `terminate` ends the run when every result of the round asks for it; `handoff` resets the context with a note (= `pi.reset` with handoff user message) (`packages/durable/README.md:168,334`; `spec.md:3665-3668`) — [[session-handoff]].
- Truncation/continuation remarks are `diagnostics`, not content; rendered as `<harness>\n[severity] message\n</harness>` (`packages/durable/src/harness/tool.ts:454-485`) — [[harness-diagnostics-channel]]. Results carry `durationMs` (`36a686ee8`).

## Constants
| name | value | path:line |
|---|---|---|
| `STRUCTURED_OUTPUT_MAX_BYTES` (bash) | 1 MiB (head/tail 512 KiB) | `packages/coding-agent/src/core/tools/bash.ts:24` |
| eval `explanation` length | 1–2000 chars | `packages/evals/evals/documentation-audit.eval.ts:18` |

## Evolution
- 2025-11 → 2026-08: results = `content` + `details` only; tools throw on failure (`9e3e319f1`).
- 2026-08-06 `1eb988cfe` (#7715) — blocked calls may terminate.
- 2026-09-29 `8562bcf66` — `outputSchema`, `structuredContent`, `isError` returns, `nestedCalls`; `1ff5b6fdd` bash 1 MiB structured output.
- 2026-10-01 `6f1072cc0` — bash output-schema description sentence ("Combined stdout and stderr, up to 1 MiB …") shortened to shrink codemode description.
- 2026-10-05 `021eae60a` (#10251) — read on images resolves to image blocks for scripts.
- 2026-10-06 `36a686ee8` — `durationMs`.

## Evidence commits
`9e3e319f1`, `1eb988cfe`, `8562bcf66`, `1ff5b6fdd`, `6f1072cc0`, `021eae60a`, `36a686ee8`.

## Quirks
- `structuredContent` is ephemeral (not persisted), so resumed sessions cannot replay script-facing values; `details` is the persisted machine channel.
- `terminate` requires unanimity; a mixed batch continues (`agent-loop.ts:688-690`).
- Bash scripts see up to 1 MiB while the model sees 50 KB — 20× asymmetry by design (07-constants).

## Failures
- (none specific; see [[hook-throw-aborts-parallel-batch]])
