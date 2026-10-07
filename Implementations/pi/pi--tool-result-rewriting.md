---
type: implementation
harness: pi
concept: tool-result-rewriting
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:862-905, packages/agent/src/types.ts:328-341, packages/coding-agent/src/core/agent-session.ts:652-724, packages/coding-agent/src/core/extensions/runner.ts:1183-1240, packages/coding-agent/src/core/extensions/types.ts:1237-1310, packages/coding-agent/src/utils/tool-result-images.ts:13-67, packages/durable/src/harness/tool.ts:343-372]
---
[[tool-result-rewriting]] in [[pi]].

## Mechanism
- **Two layers**: agent-core hook `afterToolCall(ctx, signal)` (`packages/agent/src/types.ts:328-341`) installed once by `AgentSession` (`packages/coding-agent/src/core/agent-session.ts:652-655`) which fans out to the extension `tool_result` event + image normalization (`agent-session.ts:684-724`). Callbacks read `this._extensionRunner` at execution time so `/reload` swaps runners without reinstalling hooks (`agent-session.ts:646-651`).
- **agent-core merge** (`finalizeExecutedToolCall`, `packages/agent/src/agent-loop.ts:862-905`): field-level replace of `content`, `details`, `usage`, `terminate`, `isError`; "Structured content not replaced along with the content may no longer match it" → if `content` replaced and no new `structuredContent`, structured content is **deleted** (`:884-895`). A throwing `afterToolCall` → `createErrorToolResult(message)`, `isError:true` (`:898-901`; `b9cd557d1`) — [[tool-error-as-result]].
- **Extension `tool_result` event** (`packages/coding-agent/src/core/extensions/types.ts:1237-1310,1453-1459`): input `{toolName, toolCallId, parentToolCallId?, input, content, details, structuredContent?, isError, usage}`; fires for errors too. `emitToolResult` (`packages/coding-agent/src/core/extensions/runner.ts:1183-1240`) **chains** handlers over a snapshot of handlers: each handler sees the previous patch (`currentEvent` mutated), fields applied only if defined, `content` replacement deletes `structuredContent` unless also returned; returns `undefined` if nothing modified. A throwing handler is reported via `emitError` and **skipped** (not fatal, not block) (`runner.ts:1216-1225`).
- **Image normalization after hooks**: "Runs after the extension hook so images injected or replaced by extensions are normalized too" — `normalizeToolResultImages(content, {autoResizeImages, resizeOptions: model.inputLimits.images.resize})` (`agent-session.ts:704-710`; `packages/coding-agent/src/utils/tool-result-images.ts:13-67`; `b0e05b442` #7330: "Oversized images make the provider reject the whole conversation, not just the offending turn") — [[image-normalization]].
- Nested calls (`ctx.executeTool`) go through the same `_afterToolCall(ctx, parentId)` with `parentToolCallId` set (`agent-session.ts:749-750`) — [[nested-tool-calls]].
- Uses: redaction, augmentation, error override (`isError` flip), usage attribution. Same hook surface is the permission/policy layer counterpart of [[tool-call-gate]].

## Durable variant (packages/durable)
- Tool hooks `beforeTool` (block or rewrite args) and `afterTool` (replace the result) registered via `hook(ToolTask, {...})` (`packages/durable/README.md:137,394-401`); `wrapTool()` decorates whichever same-name tool won (`README.md:170-176`).
- Settled result = tool result with retained output and last details as fallbacks, diagnostics after those reported through the api, **`afterTool` applied**, explicit text bounded with the harness truncation (`packages/durable/src/harness/tool.ts:343-372`). Any wrapper that transforms `output` must null `outputWindow` (`harness/types.ts:183-188`).

## Constants
| name | value | path:line |
|---|---|---|
| image resize cap (applied in hook) | 2000×2000, 4.5 MB base64 | `packages/coding-agent/src/utils/image-resize-core.ts:32-36` |

## Evolution
- 2025-11 → 2026-01: hook system + custom-tools wrappers; 0.24.1 hooks wrap custom tools (#248); 0.31.0 `tool_result` fires on errors (06-fixes side-door entry).
- 2026-01-05 `c6fc08453` (#454) — hooks + custom-tools merged into unified extensions.
- 2026-02-06 `2668326e0` (#1280) — chain `tool_result` patches; previously last handler won and earlier patches were lost.
- 2026-03-14 `63ac2df24` (#2113) — interception moved from per-tool wrappers into agent-core `beforeToolCall/afterToolCall` (stale sessionManager in multi-tool turns).
- 2026-04-16 `b9cd557d1` (#3084) — `afterToolCall` throws become error tool results instead of aborting the parallel batch.
- 2026-04-17 `e9808b585` (#3051) — error overrides no longer dropped (06-fixes side-door entry).
- 2026-08-03 `b0e05b442` (#7330) — images from any tool resized after hooks.
- 2026-09-29 `8562bcf66` — `structuredContent` + `parentToolCallId` added to the event; drop-on-content-replace rule.

## Evidence commits
`c6fc08453`, `2668326e0`, `63ac2df24`, `b9cd557d1`, `e9808b585`, `b0e05b442`, `8562bcf66`.

## Quirks
- Asymmetric failure policy: throwing `tool_call` handler = **block** (fail-closed), throwing `tool_result` handler = logged and skipped (fail-open) (`runner.ts:1216-1225` vs `agent-session.ts:667-679`).
- `structuredContent` is not part of persisted `ToolResultMessage` (`packages/ai/src/types.ts:607-623`; `createToolResultMessage` `agent-loop.ts:929-942`) — rewriting it only affects live programmatic callers (codemode).
- Image-processing failure keeps the original block (unlike `read`, which drops it) (`tool-result-images.ts:13-67`).

## Failures
- [[tool-result-hook-patches-lost]]
- [[hook-throw-aborts-parallel-batch]]
- [[side-door-input-bypasses-hooks]]
- [[image-content-poisoning]]
