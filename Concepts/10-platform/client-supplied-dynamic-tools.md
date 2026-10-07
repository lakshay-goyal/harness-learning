---
type: concept
stage: tools
tier: variant
aliases: [DynamicToolSpec, DynamicToolCallRequest, DynamicToolHandler, dynamic tools, item/tool/call, thread_dynamic_tools]
harnesses: [codex]
---
An embedding client (IDE, desktop app, SDK host) registers tool schemas when it starts a thread; when the model calls one, the harness forwards the call to the client over its protocol and returns the client's content items as the tool result.

## Why
- Hosts own capabilities the harness cannot reach (editor state, app UI, proprietary services) and should not need a plugin or MCP server to expose them.
- Keeps the harness process generic: the same binary serves many hosts with different tool sets.
- Calls must be cancellable with the turn, or a host that never answers hangs the agent.

## Design space
- **Where tools come from**: in-process plugin registration ([[plugin-tools]], pi) vs MCP server ([[mcp-integration]]) vs **host-declared over the session protocol** (codex).
- **Lifetime**: per thread at start (codex, persisted in `thread_dynamic_tools`) vs per turn vs dynamic add/remove.
- **Shape**: flat functions vs namespaces (codex both).
- **Exposure**: always declared vs deferrable/searchable (codex `deferLoading` → [[deferred-tool-loading]]).
- **Result content**: text only vs text + images (codex).
- **Cancellation**: oneshot await tied to turn cancellation (codex).

## Implementations
- [[codex--client-supplied-dynamic-tools|codex]] — `DynamicToolSpec::{Function, Namespace}` at thread start; app-server `item/tool/call` server→client request; result text+image items.

## Failures
- (none mined specific to the mechanism)

## Related
[[plugin-tools]] · [[mcp-integration]] · [[deferred-tool-loading]] · [[client-server-session-split]] · [[sdk-embedding]] · [[tool-schema-lowering]] · [[extensibility-model]]
