---
type: implementation
harness: pi
concept: plugin-tools
commit: b30a6dd77
files: [packages/coding-agent/src/core/extensions/types.ts:1641, packages/coding-agent/src/core/extensions/types.ts:516, packages/coding-agent/src/core/extensions/types.ts:574-650, packages/coding-agent/src/core/extensions/types.ts:383-396, packages/coding-agent/src/core/extensions/loader.ts:289-295, packages/coding-agent/docs/extensions.md:148, packages/coding-agent/docs/extensions.md:150-162, packages/coding-agent/src/core/resource-loader.ts:1246-1281]
---
[[plugin-tools]] in [[pi]].

## Mechanism
- `pi.registerTool(ToolDefinition)` (`packages/coding-agent/src/core/extensions/types.ts:1641`). Fields (`types.ts:574-650`):
  - `parameters`: TypeBox schema; must be an object schema or registration throws "must define an object parameter schema" (`loader.ts:289-295`, `acaa253cc`).
  - `execute(toolCallId, params, signal, onUpdate, ctx: ExtensionToolContext)`.
  - `renderCall` / `renderResult` / `renderShell` (TUI, reused in HTML export — see [[pi--session-export-share]]).
  - `promptSnippet` — opt-in one-liner in "Available tools" (`7817e9b22`, #2285); `promptGuidelines` → [[dynamic-tool-guidelines]].
  - `prepareArguments` — pre-validation shim ([[tool-argument-repair]]).
  - `outputSchema` + `structuredContent` ([[structured-tool-output]]).
  - `exposure`: `direct|model-only|codemode|deferred|hidden` (`types.ts:516`; semantics `docs/extensions.md:150-160`): `direct` declared+callable; `model-only` declared, never callable from tools; `codemode` callable + listed by codemode tool only; `deferred` findable via `tool_search`; `hidden` unreachable ([[deferred-tool-loading]], [[code-mode]]).
  - `namespace {name, description, instructions}` groups tools like MCP servers (`docs/extensions.md:162`).
  - MCP-style `annotations` (readOnly/destructive/idempotent/openWorld hints) → [[tool-safety-annotations]].
  - `defaultActive`, `prepareLoadout`, `executionMode: sequential|parallel` ([[parallel-tool-execution]]), `constrainedSampling` ([[constrained-tool-sampling]]).
- No unregister: re-register with `exposure:"hidden"` (`docs/extensions.md:160`). Runtime registration applies immediately without `/reload` (`packages/coding-agent/CHANGELOG.md:3296`).
- Same tool/flag name from two extensions → load error (`resource-loader.ts:1246-1281`); except `replaceable` built-ins, which drop out ([[pi--replaceable-builtin-extension]]). Extensions may override built-in tools by name (`examples/extensions/tool-override.ts`, `bash-spawn-hook.ts`, `ssh.ts`, `gondolin/` use `createBashTool` etc.).
- Nested tools: `ctx.executeTool(name,args,{signal,onUpdate})` runs through validation + `tool_call`/`tool_result` hooks; id `<parent>/<n>`; events carry `parentToolCallId`; never rejects (failures → `isError:true`); not in transcript; bounded `nestedCalls` record on the caller's result (no results; args >8 KiB per call or >32 KiB per result omitted; ≤256 calls; `complete:false` if truncated); nested `usage` rolled into caller (`types.ts:383-396`; `docs/extensions.md:148`) → [[nested-tool-calls]].
- Other registration surface (`types.ts:1641-1887`):
  - `registerCommand(name, {description, getArgumentCompletions, handler(args, ctx: ExtensionCommandContext)})` (`:1650`); duplicates across extensions invoked as `name:1`, `name:2` (`packages/coding-agent/src/core/extensions/runner.ts:807-841`, `a8a58ff26`, #1061).
  - `registerShortcut(KeyId, {handler})` (`:1653`) — see [[pi--extension-ui-primitives]]; `registerFlag(name, {type: boolean|string, default})` + `getFlag` (`:1662-1678`).
  - Actions: `sendMessage(custom, {triggerTurn, deliverAs: steer|followUp|nextTurn})` (`:1701`), `sendUserMessage(content, {deliverAs, expandPromptTemplates})` (`:1711`), `appendEntry(customType, data)` persisted NOT sent to LLM (`:1717`) → [[branch-scoped-extension-state]], `setSessionName/getSessionName/setLabel`, `exec(cmd,args)`, `getActiveTools/getAllTools/setActiveTools`, `getSettings`, `getCommands`, `setModel` (session-scoped), `get/setThinkingLevel`.
  - Providers: `registerProvider(name, ProviderConfig)` / native `Provider`; queued until runner binds, immediate after (`:1828-1829`, `loader.ts:211-216`, `975de88eb`); `unregisterProvider` (`:1844`); custom `streamSimple`, OAuth for `/login` → [[custom-provider-registration]].
  - MCP: `registerMcpServer/unregisterMcpServer/getMcpServers` (`:1864-1870`) → [[mcp-integration]]; `registerVirtualModel` routes each request to a physical model (`:1881`, `540e174c7`) → [[virtual-model-router]].
- State guidance (`docs/extensions.md:224-235`): tool-result `details` (branch-following), `appendEntry` (durable, not in context), `sendMessage` (in context), external storage; reconstruct from `ctx.sessionManager.getBranch()` in `session_start` — not from all file entries (abandoned branches are alternate histories).

### Tool-shaped example extensions
| Example | Tool | Notable |
|---|---|---|
| `hello.ts` | minimal tool | baseline (also used by docs eval) |
| `todo.ts` | todo tool + `/todos` | state in tool `details`, rebuilt on `session_start`/`session_tree` |
| `dynamic-tools.ts` | runtime registration | prompt snippets |
| `truncated-tool.ts` | rg wrapper | 50 KB / 2000-line truncation like built-ins |
| `tool-override.ts` | replaces built-in (e.g. `read`) | logging / access control |
| `structured-output.ts` | final tool | `terminate: true` skips follow-up turn |
| `question.ts` / `questionnaire.ts` | ask-user tools | select UI / tabbed multi-question |
| `tic-tac-toe.ts` | game tool | `executionMode: "sequential"` demo |
| `subagent/` | single/parallel/chain sub-agents | spawns `pi` subprocesses ([[subagent-as-subprocess]]) |
| `reload-runtime.ts`, `shutdown-command.ts` | tools that reload / quit | `ctx.reload()`, `ctx.shutdown()` |
| `bash-spawn-hook.ts`, `ssh.ts`, `gondolin/`, `sandbox/` | overridden built-in tools | `createBashTool` with custom ops ([[pluggable-tool-backends]], [[tool-only-isolation]]) |
| `tools.ts` | `/tools` enable/disable | `setActiveTools` with persistence |
Full catalogue with hooks: [[pi--extension-event-hooks]].

## Constants
| name | value | path:line |
|---|---|---|
| nested-call record cap | 256 calls | `docs/extensions.md:148` |
| nested arg omission | >8 KiB per call / >32 KiB per result | `docs/extensions.md:148` |
| exposure tiers | 5 | `types.ts:516` |

## Evolution
- 2025-12-17 `e7097d911` custom tools as a separate system with session lifecycle (`--tool`).
- 2026-01-05 `c6fc08453` custom tools merged into unified extensions (`registerTool`, `-e`) (#454); `core/custom-tools/*` deleted.
- 2026-02-18 `975de88eb` `registerProvider` flushed immediately after `bindCore`, `unregisterProvider` added.
- 2026-03-17 `7817e9b22` prompt snippets opt-in (#2285).
- 2026-03-23 `a8a58ff26` duplicate slash commands disambiguated (#1061).
- 2026-09-09 `acaa253cc` reject tools without object parameter schema (`packages/coding-agent/CHANGELOG.md:472`).
- 2026-09-28 `540e174c7` virtual models (#10035); 2026-09-29 `8562bcf66` exposure tiers used by built-in codemode/tool_search/MCP.
- 2026-10-03 `11449730c` tool renderer resolver can render MCP tools before their server connects.
- 2026-10-06 `b0114ef5f` (pi-durable) `models` exposed on `ToolExecutionApi` and `HookApi`, so durable tools/hooks can call models.

## Evidence commits
`e7097d911` `c6fc08453` `975de88eb` `7817e9b22` `a8a58ff26` `acaa253cc` `540e174c7` `8562bcf66` `11449730c`

## Quirks
- `tool_call` hooks can mutate input without re-validation (see [[pi--extension-event-hooks]]).
- Removing a tool requires `exposure:"hidden"`; there is no unregister (`docs/extensions.md:160`).
- Extension tools run on the host even inside container/VM routing unless they delegate (`docs/containerization.md:16,181-183`) → [[tool-only-isolation]].

## Durable variant (packages/durable)
- `defineTool` + TypeBox validation; intent committed before execute; per-tool `replay: safe|unsafe` (default unsafe) → [[crash-safe-tool-replay]]; `beforeTool`/`afterTool` hooks; `wrapTool`; later extension overrides same-name tool (`packages/durable/src/harness/tool.ts:46-110`; `packages/durable/README.md:147-178`).
- A call to a tool the request did not offer → `tool_unavailable` result with no task (`spec.md:3576-3594`).

## Failures
[[plugin-tool-without-schema-breaks-requests]] · [[side-door-input-bypasses-hooks]]
