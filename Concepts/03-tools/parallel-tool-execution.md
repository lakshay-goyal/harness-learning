---
type: concept
stage: tools
tier: candidate
aliases: ["toolExecution: parallel", "executionMode: sequential", parallel-tool-execution-default, executeToolCallsParallel, ToolCallRuntime, parallel_tool_calls, supports_parallel_tool_calls]
harnesses: [pi, codex]
---
Execute all tool calls of one assistant turn concurrently — after a sequential preflight (validation, policy hooks, approvals) — emitting completion events as they finish but appending results in call order, unless a tool opts out.

## Why
- Independent reads/searches/commands in one turn otherwise serialize wall-clock time.
- Concurrency creates races on shared resources (same-file edits clobbered, [[concurrent-file-mutation-interleave]]) and on human-in-the-loop tools ([[interactive-tool-in-parallel-batch]]).
- Completion order ≠ transcript order; persisting in completion order breaks call/result pairing expectations ([[parallel-tool-results-order]]).
- An exception in one branch must not take down the batch ([[hook-throw-aborts-parallel-batch]]).

## Design space
- Sequential always (pi pre-2026-03).
- **Parallel by default, sequential preflight, ordered results** (pi).
- Per-tool opt-out flag; pi: any sequential tool in a batch serializes the whole batch.
- Resource-level serialization instead of opting out (pi: [[per-file-mutation-queue]] for edit/write).
- Read-only/mutating classification to decide concurrency (not in pi; annotations exist only for MCP → [[tool-safety-annotations]]).
- Parallelism inside one tool call via a script ([[code-mode]]) instead of across calls.
- **Per-tool reader/writer lock inside one batch** (✔ codex: parallel-safe tools share a read lock, all others take the exclusive write lock) vs whole-batch parallel/sequential (✔ pi).
- Tools start while the response is still streaming; results appended in call order after the stream (✔ codex `FuturesOrdered`).
- Default not parallel, per-tool opt-in (✔ codex: exec_command, write_stdin, view_image, tool_search, MCP resources) vs parallel by default (✔ pi).
- Read-only classification from MCP annotations, distrusted while the catalog is cached (✔ codex `readOnlyHint`).
- Shell commands run in parallel without any file lock (✔ codex) → [[per-file-mutation-queue]] absent.
- Approvals keyed by call id so one approval can't cover a parallel batch (✔ codex `c4b771a16f`).

## Implementations
- [[pi--parallel-tool-execution|pi]] — `toolExecution` default "parallel"; `Promise.all` over prepared thunks; `tool_execution_end` in completion order, results in source order; `executionMode:"sequential"` per tool.
- [[codex--parallel-tool-execution|codex]] — `parallel_tool_calls: true` always (except Responses Lite); per-turn `RwLock<()>` gate; tools spawned during streaming, drained in order.

## Failures
- [[parallel-tool-results-order]]
- [[hook-throw-aborts-parallel-batch]]
- [[interactive-tool-in-parallel-batch]]
- [[concurrent-file-mutation-interleave]]
- [[unannotated-mcp-tools-serialized]]
- [[parallel-tools-reveal-latent-bugs]]
- (07) [[approval-scope-too-broad-in-parallel-batch]]
- [[steering-skips-pending-tool-calls]] (01-loop) — A user message typed mid-run caused the remaining tool calls of the current assistant message to be skipped…
- [[tool-preflight-ignores-abort]] (01-loop) — After abort, tools kept going: an extension's ctx.abort() during one tool's confirmation dialog left sibling…
- [[pre-tool-hook-sees-stale-state]] (07-safety) — In multi-tool turns, tool_call handlers (policy/gate extensions) read ctx.sessionManager state that did not…

## Related
[[turn-loop]] · [[per-file-mutation-queue]] · [[tool-error-as-result]] · [[nested-tool-calls]] · [[abort-propagation]] · [[steering-queue]] · [[tool-call-gate]] · [[code-mode]] · [[tool-safety-annotations]] · [[mcp-integration]]
