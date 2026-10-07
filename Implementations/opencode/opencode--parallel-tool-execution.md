---
type: implementation
harness: opencode
concept: parallel-tool-execution
commit: ecc4916b5a
files: [packages/opencode/src/session/tools.ts:41-134, packages/opencode/src/session/llm/request.ts:184, packages/core/src/session/runner/llm.ts:141-142, packages/core/src/session/runner/llm.ts:184, packages/core/src/session/runner/llm.ts:259-280, packages/codemode/src/stdlib/promise.ts:6, packages/opencode/src/session/prompt/gpt.txt:6, packages/opencode/src/session/prompt/trinity.txt:84]
---
[[parallel-tool-execution]] in [[opencode]].

## Mechanism
### Legacy runtime
- The AI SDK owns dispatch: each registry tool is an AI SDK tool whose `execute` runs inside `streamText`; parallel calls run concurrently as the SDK schedules them (`packages/opencode/src/session/tools.ts:41-134`). No harness preflight phase; permission asks happen inside each execute via `ctx.ask` → [[permission-ruleset]].
- No harness-level per-tool sequential flag; same-file safety only via edit's per-path lock → [[per-file-mutation-queue]].
- Tool declarations sorted by name for a stable prefix (`packages/opencode/src/session/llm/request.ts:184`, `83bb216486` 2026-05-08).
- Parallelism is prompt-driven per model: gpt.txt "Use `multi_tool_use.parallel` to parallelize tool calls and only this" (`packages/opencode/src/session/prompt/gpt.txt:6`); trinity.txt "Use exactly one tool per assistant message" (`trinity.txt:84`) → [[model-cannot-parallel-tool-call]], [[per-model-system-prompt]].
### v2 runtime
- **Eager settlement while the stream is open**: each completed `tool-call` is durably published, then forked into a `FiberSet` (`packages/core/src/session/runner/llm.ts:184,259-280`); settle + publish wrapped in `uninterruptibleMask`; publication serialized by a 1-permit semaphore.
- After stream close `awaitToolFibers` races `FiberSet.join` with `awaitEmpty` (`llm.ts:141-142`).
- "Eager local-tool execution is intentionally unbounded in the current local slice" — no concurrency cap, no per-turn call limit (`specs/v2/session.md:173`).
- Settlement keyed by assistant message ID because provider call IDs repeat across turns → [[tool-call-id-collision]].
### Code mode
- Inside an `execute` script, `Promise.all` runs ≤ 8 tool calls concurrently (`packages/codemode/src/stdlib/promise.ts:6`) → [[code-mode]].

## Constants
| name | value | path:line |
|---|---|---|
| v2 eager concurrency | unbounded | `specs/v2/session.md:173` |
| `TOOL_CALL_CONCURRENCY` (codemode) | 8 | `packages/codemode/src/stdlib/promise.ts:6` |

## Evolution
- 2025-11-15 `1056b36eae` experimental `batch` meta-tool (1–25 parallel calls); deleted 2026-04-07 `463318486f` → [[minimal-default-toolset]].
- 2026-01-28 `558590712d` parallel reads double-loaded AGENTS.md → [[context-file-loaded-twice-in-worktrees]].
- 2026-04-20 `8bc4f91fd9` parallel edits overrode each other → [[concurrent-file-mutation-interleave]].

## Quirks / drift
- Interactive `question` can be issued in a parallel batch (no gate found) → [[interactive-tool-in-parallel-batch]] (unverified in practice).

Contrast: pi validates and gates sequentially, then executes in parallel with ordered results → [[pi--parallel-tool-execution|pi]].
