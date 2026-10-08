---
type: concept
stage: architecture
tier: must-have
aliases: ["registerTool", "ToolDefinition", "extension-custom-tools", "custom tools (pre-2026)", "{tool,tools}/*.ts", "tool plugin hook"]
harnesses: [pi, opencode]
---
Plugin-registered model-callable tools: schema, executor, renderers, prompt contribution and exposure level declared through the same API the harness uses for its own tools.

## Why
- Without it every capability must ship in core; with it, sub-agents, todo, questions, structured output, MCP and sandboxed tools become plugins (see example table in [[pi--plugin-tools]]).
- A plugin tool without a valid schema breaks every provider request, not just its own call ([[plugin-tool-without-schema-breaks-requests]]).
- Plugin tools must flow through the same validation and hook pipeline, otherwise they bypass policy ([[side-door-input-bypasses-hooks]]).

## Design space
- **Registration time**: load-time only vs runtime (pi: runtime registration applies immediately, no reload).
- **Removal**: unregister vs hide (pi: no unregister; re-register `exposure:"hidden"`).
- **Exposure tiers**: always declared vs deferred/searchable vs code-mode-only vs hidden (pi: `direct|model-only|codemode|deferred|hidden`) — see [[deferred-tool-loading]], [[code-mode]].
- **Schema system**: JSON Schema vs TypeBox (pi) vs zod; strictness / constrained sampling opt-in ([[constrained-tool-sampling]]).
- **Prompt contribution**: tool always listed vs opt-in snippet + guidelines (pi `promptSnippet`, `promptGuidelines`, see [[dynamic-tool-guidelines]]).
- **Rendering**: harness renders generically vs plugin supplies call/result renderers (pi) — see [[extension-ui-primitives]].
- **Nested calls**: tools may call other tools through the full pipeline (pi `ctx.executeTool`, [[nested-tool-calls]]).
- **Name conflicts**: load error (pi, between plugins) vs override (pi, same-name override of built-ins, `tool-override.ts`; `replaceable` built-ins step aside).
- **Execution mode**: per-tool sequential opt-out from parallel batches ([[parallel-tool-execution]]).
- codex: absent as in-process plugin registration — plugin bundles cannot carry executable tools (`codex-rs/plugin/src/manifest.rs:8-58`, [[no-executable-plugins]]); third-party tools arrive via MCP servers declared in bundles ([[mcp-integration]]) or are declared by the embedding client over the protocol ([[client-supplied-dynamic-tools]]); first-party tools come from compiled `codex-rs/ext/*` crates.
- **Drop-in discovery**: every config dir's `tool(s)/*.{js,ts}` exports become tools (opencode).
- **Schema system**: Zod `args` converted to JSON Schema, missing `args` normalized to `{}` (opencode) vs reject at registration (pi).
- **Stale-call rejection**: per-turn registration identity; a call whose registration changed settles as "Stale tool call" (opencode v2).

## Implementations
- [[pi--plugin-tools|pi]] — `pi.registerTool(ToolDefinition)` with TypeBox params, renderers, exposure, annotations, `prepareArguments`, `outputSchema`, nested `executeTool`.
- [[opencode--plugin-tools|opencode]] — plugin `tool` hook + `{tool,tools}/*.{js,ts}` files; Zod → JSON Schema; `tool.definition` rewrite hook; v2 scope-bound overlay registrations.

## Failures
- [[plugin-tool-without-schema-breaks-requests]]
- [[side-door-input-bypasses-hooks]]
- [[tool-wrapper-accumulation]]
- [[untrusted-repo-loads-executable-config]]

## Related
[[extension-event-hooks]] · [[runtime-plugin-loading]] · [[replaceable-builtin-extension]] · [[tool-safety-annotations]] · [[structured-tool-output]] · [[tool-argument-repair]] · [[mcp-integration]] · [[pluggable-tool-backends]] · [[minimal-default-toolset]] · [[extensibility-model]]

## Tradeoffs
- [[todo-tool-vs-none]]
