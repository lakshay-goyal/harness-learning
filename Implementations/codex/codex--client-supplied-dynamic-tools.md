---
type: implementation
harness: codex
concept: client-supplied-dynamic-tools
commit: 622e9e3696
files: [codex-rs/protocol/src/dynamic_tools.rs:1, codex-rs/core/src/tools/spec_plan.rs:1507, codex-rs/core/src/tools/handlers/dynamic.rs:125, codex-rs/app-server-protocol/src/protocol/common.rs:1516]
---
[[client-supplied-dynamic-tools]] in [[codex]].

## Mechanism
- **Spec**: `Function{name, description, inputSchema, deferLoading}` or `Namespace{name, description, tools}` (`codex-rs/protocol/src/dynamic_tools.rs:1-40`); registered into the tool plan by `append_dynamic_tool_runtimes` (`codex-rs/core/src/tools/spec_plan.rs:1507`); foreign schemas pass through the sanitizer/compactor ([[tool-schema-normalization]]).
- **Deferral**: `deferLoading` tools are searchable via `tool_search` (`6991be7ead` 2026-04-18) → [[deferred-tool-loading]].
- **Execution**: handler emits a request event and awaits a oneshot response, cancellable with the turn ("dynamic tool call was cancelled before receiving a response") (`codex-rs/core/src/tools/handlers/dynamic.rs:125-160`, `:190-240`); core op `DynamicToolResponse` (`codex-rs/core/src/session/handlers.rs:492-720`); app-server server→client request `item/tool/call` (`codex-rs/app-server-protocol/src/protocol/common.rs:1516-1686`); thread item `DynamicToolCall`.
- **Output**: text + image content items (`5ea107a088` 2026-02-04).
- **Persistence**: per-thread registration mirrored in SQLite `thread_dynamic_tools` table ([[codex--sqlite-session-index|sqlite-session-index]]).

## Evolution
- 2026-01-26 `d594693d1a` "dynamic tools injection".
- 2026-02-04 `5ea107a088` text + image output.
- 2026-04-18 `6991be7ead` deferrable dynamic tools.

## Versus pi
- pi has no host-declared tool channel; equivalent capabilities come from in-process `pi.registerTool` plugins ([[pi--plugin-tools]]) or MCP.
