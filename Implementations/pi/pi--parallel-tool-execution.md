---
type: implementation
harness: pi
concept: parallel-tool-execution
commit: b30a6dd77
files: [packages/agent/src/agent.ts:253, packages/agent/src/types.ts:40, packages/agent/src/types.ts:491, packages/agent/src/agent-loop.ts:515, packages/agent/src/agent-loop.ts:586, packages/agent/src/agent-loop.ts:646, packages/coding-agent/src/core/nested-tool-calls.ts:202, packages/durable/src/harness/agent.ts:68]
---
[[parallel-tool-execution]] in [[pi]].

## Mechanism
- Agent option `toolExecution` default `"parallel"` (`packages/agent/src/agent.ts:253`; doc `types.ts:305-317`); coding-agent `sdk.ts` never sets it (`sdk.ts:411-446`) → interactive pi runs parallel.
- Per-tool `executionMode?: "sequential"|"parallel"` (`packages/agent/src/types.ts:491-498`, `bfa11a50e` #3345). If `config.toolExecution === "sequential"` **or any call in the batch** targets a sequential tool, the **whole batch** runs sequentially (`agent-loop.ts:515-523`).
- **Parallel path** (`agent-loop.ts:586-660`):
  1. Preflight **sequentially** for all calls: `tool_execution_start` → `prepareToolCall` (unknown-tool check, `prepareArguments`, validation, `beforeToolCall`/extension `tool_call` hook which may prompt the human) → abort re-check after each (`b94482762` #4276).
  2. Prepared calls become thunks; immediate outcomes (errors/blocks) stay values.
  3. `Promise.all` runs thunks concurrently; each thunk checks abort first → "Operation aborted" (`:623`, `afda4d620` #8936).
  4. `tool_execution_end` emitted in **completion order** as each finalizes (`759d55152` #3503).
  5. Tool-result messages emitted/persisted after all finish in **assistant source order** (`:646-654`; contract `types.ts:36-47`).
  6. `terminate` = every finalized result has `terminate:true` (`shouldTerminateToolBatch` `:690-692`) → see [[structured-tool-output]].
- **Sequential path** (`agent-loop.ts:530-584`): per call start → prepare → execute → finalize → end → result message; `break` if signal aborted after a call (`:575-577`).
- Barrier: listeners are awaited, so assistant `message_end` (incl. session persistence) completes before preflight starts; `beforeToolCall` sees state containing the assistant message (`packages/agent/README.md`; `agent.ts:565-612`; `63ac2df24` #2113 fixed stale sessionManager in multi-tool turns).
- Progress: `onUpdate` → `tool_execution_update`; late updates after settle ignored via `acceptingUpdates` (`agent-loop.ts:826-857`, `daab056ac` #5573). `durationMs` per tool via `performance.now()` excluding hooks (`36a686ee8`).
- **Same-file safety without opting out**: no built-in tool sets `executionMode` (grep of `executionMode` in `CA` hits only wrapper/types/nested-tool-calls); edit/write rely on [[per-file-mutation-queue]] instead.
- **Opt-out users**: example `question.ts` tool (human Q&A) → `executionMode:"sequential"` (`ec857fece` #6189; `examples/extensions/question.ts:50`); `tic-tac-toe.ts` demo.
- **Nested calls** respect sequential tools: exclusive nested calls go through a promise-chain `queueTail`; `holdsQueue` re-entrancy prevents a sequential caller's own nested calls from deadlocking (`packages/coding-agent/src/core/nested-tool-calls.ts:146-152,202-218`) → [[nested-tool-calls]].
- Codemode scripts can parallelize inside one tool call (`Promise.allSettled` guideline, `packages/coding-agent/src/extensions/codemode/tool.ts:127-132`) → [[code-mode]].

## Constants
| name | value | path:line |
|---|---|---|
| `toolExecution` default | `"parallel"` | `packages/agent/src/agent.ts:253` |
| durable `toolExecution` default | `"parallel"` | `packages/durable/src/harness/agent.ts:68` |

## Evolution
- Early: sequential; steering polled between tool calls aborted remaining calls with fake "Skipped due to queued user message" (`117af076c`) → `208a2cc12` defer steering until batch completes ([[steering-queue]]).
- 2026-03-14 `63ac2df24` (#2113) parallel default + sequential preflight; interception moved into agent-core hooks.
- 2026-03-20 `74a46fc7e` (#2327) file mutation queue (parallel edits clobbered each other).
- 2026-04-16 `b9cd557d1` (#3084) hook throw no longer aborts batch.
- 2026-04-18 `bfa11a50e` (#3345) per-tool `executionMode`.
- 2026-04-22 `759d55152` (#3503) eager completion events, source-ordered results.
- 2026-05-19 `b94482762` (#4276) abort re-check between preflights.
- 2026-06-12 `daab056ac` (#5573) late progress ignored.
- 2026-07-02 `ec857fece` (#6189) question example sequential.
- 2026-09-01 `afda4d620` (#8936) prepared thunks honor preflight abort.

## Evidence commits
`117af076c`, `208a2cc12`, `63ac2df24`, `74a46fc7e`, `b9cd557d1`, `bfa11a50e`, `759d55152`, `b94482762`, `daab056ac`, `ec857fece`, `afda4d620`, `36a686ee8`

## Quirks
- One slow tool delays **all** result messages and therefore the next model request (inferred from `Promise.all` + source-order emission).
- Sequential is contagious per batch: a single sequential tool serializes unrelated parallel-safe siblings.
- Interactive approvals happen in preflight, serially, before any tool runs — a user sees all confirmations up front.
- Mutation queue protects edit/write only; `bash` writing the same file concurrently is unguarded ("Not a lock against `bash` or other processes", `packages/durable/src/tools/file-mutation-queue.ts:28-32`).

## Durable variant (packages/durable)
- Tool rounds parallel by default (`packages/durable/src/harness/agent.ts:68`); a round is sequential if settings say so or any called tool has `executionMode:"sequential"`; sequential rounds create one task at a time; generation waits on tool tasks with `allSettled` (`spec.md:3536-3594,3651-3681`). Each call is its own durable `pi.tool` task → [[crash-safe-tool-replay]].

## Failures
- [[parallel-tool-results-order]]
- [[hook-throw-aborts-parallel-batch]]
- [[interactive-tool-in-parallel-batch]]
- [[concurrent-file-mutation-interleave]]
