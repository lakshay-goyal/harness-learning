---
type: concept
stage: tools
tier: must-have
aliases: [ctx.executeTool, NestedToolCallRunner, parentToolCallId, NESTED_CALL_LIMITS, nestedCalls, nested-tool-calls-through-pipeline, ToolCallSource::CodeMode]
harnesses: [pi, opencode, codex]
---
Tools invoking other tools programmatically through the same validation, policy-hook and permission pipeline as model calls, with provenance (parent id) and a bounded record attached to the parent result.

## Why
- An orchestrating tool (code-mode script, macro tool) that bypasses the pipeline is a side door around permission hooks ([[side-door-input-bypasses-hooks]]).
- Unbounded nested call records bloat the transcript; nested results themselves shouldn't be persisted twice.
- Sequential-only tools called from inside a sequential caller can deadlock on the serialization queue.

## Design space
- Direct function calls to tool implementations (no hooks) vs **through the full pipeline** (pi `ctx.executeTool`).
- Record nested calls: none vs bounded summary on parent result (pi: ≤256 calls, 8KiB args each, 32KiB total, 500 error chars) vs full transcript entries.
- Ids: hierarchical `<parent>/<n>` (pi).
- Serialization of sequential tools via a global queue with re-entrancy (pi `holdsQueue`).
- Agent-level delegation (subagents) as the heavier alternative → [[subagent-as-subprocess]], [[task-owned-subagent]].
- Nesting only from code-mode cells; provenance = cell id + per-cell runtime call id (✔ codex) vs any tool via `ctx.executeTool` (✔ pi).
- Exposure-level exclusion of tools from scripts (`DirectModelOnly`: question tool, context reset) (✔ codex).
- Nested calls restricted to one tool class (opencode: MCP only, built-ins unreachable from scripts).

## Implementations
- [[pi--nested-tool-calls|pi]] — `ctx.executeTool(name,args)` → `runToolCall` with hooks, `parentToolCallId` events, bounded `nestedCalls` record + summed usage on the parent toolResult; used by codemode.
- [[codex--nested-tool-calls|codex]] — code-mode `tools.*` calls re-enter the same router (`ToolCallSource::CodeMode`), same RwLock gate, hooks, approvals.
- [[opencode--nested-tool-calls|opencode]] — only from code mode: MCP child calls run `tool.execute.before` → permission ask → call → `tool.execute.after`; no call budget, no nested-call record.

## Failures
- related (07): [[side-door-input-bypasses-hooks]]

## Related
[[code-mode]] · [[tool-call-gate]] · [[parallel-tool-execution]] · [[structured-tool-output]] · [[file-op-tracking]] · [[plugin-tools]] · [[deferred-tool-loading]]
