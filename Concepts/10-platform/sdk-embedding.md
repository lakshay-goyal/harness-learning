---
type: concept
stage: architecture
tier: candidate
aliases: ["createAgentSession", "AgentSession", "AgentSessionRuntime", "DefaultResourceLoader", "SessionManager.inMemory()", "createAgentSessionServices", "pi SDK", "@opencode-ai/sdk", createOpencode, "@opencode-ai/sdk-next"]
harnesses: [pi, opencode]
---
In-process library API exposing the full agent session (prompt/steer/follow-up/abort/subscribe) with every boundary — models, settings, session store, resources, tools — injectable.

## Why
- Lets other programs (bots, IDE back-ends, eval harnesses, sub-agent runners) reuse the exact CLI agent without a subprocess.
- Testing and evals need in-memory, credential-isolated, plugin-controlled sessions (pi's own eval harness is an SDK consumer, see [[harness-evals]]).
- Session replacement (new/switch/fork) invalidates subscriptions — embedding hosts must rebind ([[stale-plugin-context-after-session-replacement]]).

## Design space
- **Surface**: thin "run prompt" helper vs full session object with lifecycle (pi) vs low-level loop only (pi `packages/agent` `Agent` also exported).
- **Injection points**: model runtime, settings, session store (file vs in-memory), resource loader (discovery vs inline factories), tool allow/deny lists (pi: all).
- **Defaults parity with CLI**: same built-ins vs explicit opt-in (pi: SDK does not load codemode/tool_search/MCP built-ins).
- **Busy-session prompt**: implicit queue vs reject unless steer/follow-up chosen (pi rejects).
- **Session replacement**: mutate in place vs replace object + rebind (pi `AgentSessionRuntime`).
- **Process isolation alternative**: [[headless-rpc-mode]].
- **Out-of-process SDK**: OpenAPI-generated HTTP client that spawns the CLI server (opencode legacy `createOpencode`).
- **Same public API executed in memory**: no listener, server middleware preserved (opencode v2 Embedded OpenCode).

## Implementations
- [[pi--sdk-embedding|pi]] — `createAgentSession()` → `AgentSession`; `AgentSessionRuntime` for new/switch/fork/import; 14 typechecked SDK examples.
- [[opencode--sdk-embedding|opencode]] — `@opencode-ai/sdk` = hey-api client + spawned `opencode serve`; v2 `@opencode-ai/sdk-next` runs the server router in memory.

## Failures
- [[stale-plugin-context-after-session-replacement]]

## Tradeoffs
- [[client-server-vs-single-process]]

## Related
[[headless-rpc-mode]] · [[agent-event-stream]] · [[runtime-plugin-loading]] · [[layered-settings]] · [[session-tree]] · [[harness-evals]] · [[replaceable-builtin-extension]] · [[steering-queue]] · [[follow-up-queue]] · [[client-server-session-split]]
