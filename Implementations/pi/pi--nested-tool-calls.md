---
type: implementation
harness: pi
concept: nested-tool-calls
commit: b30a6dd77
files: [packages/coding-agent/src/core/nested-tool-calls.ts:1-245, packages/coding-agent/src/core/agent-session.ts:725-764, packages/coding-agent/src/core/agent-session.ts:1103-1115, packages/coding-agent/src/core/agent-session.ts:1557-1566, packages/coding-agent/src/core/extensions/types.ts:376-395, packages/ai/src/types.ts:600-618]
---
[[nested-tool-calls]] in [[pi]].

## Mechanism
- **API** `ctx.executeTool(name, args, {signal, onUpdate})` on `ExtensionToolContext` (`packages/coding-agent/src/core/extensions/types.ts:376-395`): "The call gets the id `<calling id>/<n>`, and the `tool_call`, `tool_result`, and `tool_execution_*` events carry `parentToolCallId`. It does not appear in the transcript; a bounded record of it is kept as `nestedCalls` on the calling tool's result message. Never rejects for tool failures: unknown tools, validation errors, blocked calls, and thrown errors come back as `isError: true`." `ctx.tools` = what it can call.
- Callable set = active `direct` tools + every registered `codemode`/`deferred` tool (`packages/coding-agent/src/core/agent-session.ts:1557-1566`); `model-only` (codemode, tool_search) never callable → no script-starts-script recursion ([[pi--deferred-tool-loading|exposure tiers]]).
- **Same pipeline** (`_executeNestedToolCall`, `agent-session.ts:725-764`): lazily builds `NestedToolCallRunner`; each call → agent-core `runToolCall(toolCall, {tools: callable, assistantMessage: last assistant, context, beforeToolCall: _beforeToolCall(ctx, parentId), afterToolCall: _afterToolCall(ctx, parentId), signal, onUpdate})` — i.e. `prepareArguments` → schema validation → extension `tool_call` gate (fail-closed) → execute → `tool_result` patches + image normalization. If no assistant message exists → error "No assistant message issued this call" (`:739-745`). Permission extensions therefore see every codemode/MCP sub-call ([[tool-call-gate]], [[tool-result-rewriting]]).
- **Runner** (`packages/coding-agent/src/core/nested-tool-calls.ts`; header `:1-9` "The agent loop does not know about them … Nothing here runs until a tool calls `ctx.executeTool()`"):
  - Scope per caller id: `{recorder, nextId, holdsQueue}`; child id `${callerId}/${n}` (`:182-192`); nested calls can nest further (scope registered for the child id, `:214-218`).
  - Events `tool_execution_start/update/end` with `parentToolCallId`, emitted to extensions and session listeners (`:194-200,224-245`; `agent-session.ts:757-760`).
  - **Sequential exclusivity**: a call is `exclusive` if agent `toolExecution === "sequential"` or target tool `executionMode === "sequential"` and the scope doesn't already hold the queue; exclusive calls chain on a global promise `queueTail`; `holdsQueue` re-entrancy lets a sequential caller's own nested calls proceed without deadlock (`:146-152,166,202-218`) — [[parallel-tool-execution]].
  - Usage: "Nested results are not persisted, so their usage is only counted through the recorder" (`:237-238`).
- **Record** (`NESTED_CALL_LIMITS`, `nested-tool-calls.ts:20-31`): arguments over per-call or total size omitted, calls beyond count dropped, record marked `complete:false` — maxCalls 256, 8 KiB args/call, 32 KiB args total, 500 error chars (`:57-84`).
- **Attach**: on the parent's toolResult `message_start`, `takeRecord(toolCallId)` → `message.nestedCalls = calls`, usage combined into `message.usage` (`agent-session.ts:1103-1115`); cleared on `agent_end`. Type: `NestedToolCalls {calls, complete}` — "Calls this tool made to other tools. Kept for the session record; not sent to the model" (`packages/ai/src/types.ts:600-618`).
- **Downstream consumers**: compaction file-op tracking also reads `toolResult.nestedCalls` so reads/edits made from codemode scripts land in `<read-files>/<modified-files>` (`packages/coding-agent/src/core/compaction/utils.ts:33`) — [[file-op-tracking]]; session cost includes nested usage (`9a100c7cc`).
- Not agent-level delegation: the substrate for tool-level orchestration (codemode) instead of subagents (`docs/extensions.md:148`; [[no-subagents-core]]).

## Constants
| name | value | path:line |
|---|---|---|
| `NESTED_CALL_LIMITS.maxCalls` | 256 | `packages/coding-agent/src/core/nested-tool-calls.ts:27` |
| `maxArgumentBytesPerCall` | 8 KiB | `nested-tool-calls.ts:28` |
| `maxArgumentBytesTotal` | 32 KiB | `nested-tool-calls.ts:29` |
| `maxErrorChars` | 500 | `nested-tool-calls.ts:30` |

## Evolution
- 2026-03-14 `63ac2df24` (#2113) — tool interception moved into agent-core `beforeToolCall/afterToolCall` hooks; prerequisite for reusing `runToolCall` for nested calls.
- 2026-09-29 `8562bcf66` — `ctx.executeTool`, `NestedToolCallRunner`, `parentToolCallId`, `nestedCalls` land with codemode/MCP.
- 2026-09-29 `9a100c7cc` — nested tool + classifier usage reported in session cost.
- 2026-10-06 `36a686ee8` — `durationMs` on results (applies to nested outcomes too).

## Evidence commits
`63ac2df24`, `8562bcf66`, `9a100c7cc`, `36a686ee8`.

## Quirks
- Nested results themselves are never persisted — only a bounded record (name, args ≤8 KiB, status, error ≤500 chars); after the fact you cannot see what a script's sub-call returned.
- `assistantMessage` passed to hooks is the *last* assistant message, not the call that spawned the script (`agent-session.ts:738`).
- The global `queueTail` serializes exclusive nested calls across *all* parents, not per parent.
- Durable harness has no `ctx.executeTool` equivalent at HEAD (grep `executeTool` in `packages/durable/src` → no hits).

## Failures
- (none specific; see [[codemode-sandbox-escape-to-host]], [[side-door-input-bypasses-hooks]])
