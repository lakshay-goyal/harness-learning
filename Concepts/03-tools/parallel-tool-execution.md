---
type: concept
stage: tools
tier: candidate
aliases: ["toolExecution: parallel", "executionMode: sequential", parallel-tool-execution-default, executeToolCallsParallel, FiberSet, "eager local-tool execution"]
harnesses: [pi, opencode]
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
- Eager start while the provider stream is still open (opencode v2) vs after the assistant message completes (pi).
- Parallelism via a meta-tool taking N calls (opencode `batch`, removed) vs native parallel calls.
- Model-conditional prompting: GPT "parallelize", Trinity "one tool per message" (opencode) → [[per-model-system-prompt]].

## Implementations
- [[pi--parallel-tool-execution|pi]] — `toolExecution` default "parallel"; `Promise.all` over prepared thunks; `tool_execution_end` in completion order, results in source order; `executionMode:"sequential"` per tool.
- [[opencode--parallel-tool-execution|opencode]] — legacy: AI SDK runs calls concurrently, no harness preflight; v2: each completed call forked into a fiber while the stream is still open, "intentionally unbounded"; `batch` meta-tool removed.

## Failures
- [[tool-preflight-ignores-abort]]
- [[parallel-tool-results-order]]
- [[hook-throw-aborts-parallel-batch]]
- [[interactive-tool-in-parallel-batch]]
- [[concurrent-file-mutation-interleave]]
- [[model-cannot-parallel-tool-call]]

## Related
[[turn-loop]] · [[per-file-mutation-queue]] · [[tool-error-as-result]] · [[nested-tool-calls]] · [[abort-propagation]] · [[steering-queue]] · [[tool-call-gate]] · [[code-mode]]
