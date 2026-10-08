---
type: implementation
harness: pi
concept: crash-safe-tool-replay
commit: b30a6dd77
files: [packages/durable/docs/spec.md:1872-1883, packages/durable/src/harness/tool.ts:37-110, packages/durable/README.md:168, packages/coding-agent/src/experimental/durable/subagent.ts:24-55, packages/coding-agent/src/experimental/vacation/vacation.ts:16-55, packages/agent/src/types.ts:489-490]
---
[[crash-safe-tool-replay]] in [[pi]] — exists only in the durable harness (`packages/durable`, "Pico5"); stable coding-agent (`packages/agent` loop + JSONL sessions) has **no** equivalent: a crash mid-tool loses the in-flight call; on resume the orphaned tool call is repaired at the provider boundary ([[transcript-replay-repair]]).

## Mechanism
- **Effect sandwich** (`packages/durable/docs/spec.md:1872-1883`, §5.2): "commit intent phase → perform external effect → commit outcome or next phase. Reopening in an intent phase means the effect may have happened. The phase handler retries safely, polls an external handle, or records interruption. Deferred providers are represented by a durable phase containing their handle and next poll time." Generic for all durable tasks ([[durable-execution]]).
- **Tool task `pi.tool`** (`packages/durable/src/harness/tool.ts:50-110`), checkpoint `{phase:"call"} | {phase:"execute"; arguments; replay:"safe"|"unsafe"}` (`tool.ts:37-38`):
  - `call` phase: resolve tool from the phase agent (missing → `tool_unavailable` result, no execution); `prepareArguments` + validate; `beforeTool` hook chain (block or rewrite args; hook throw = block unless aborting); **re-validate**; commit intent `{phase:"execute", arguments: final, replay: tool.replay ?? "unsafe"}` with slot status `running` (`tool.ts:55-90`); then executes in the same handler — "nothing separates resolution from" execution (`tool.ts:46-48`).
  - `execute` phase "is reached only by recovery and applies the replay rule" (`tool.ts:48`): rerun **only when both the stored and the current policy are `safe`** (`tool.ts:93-98`) — clears progress the interrupted attempt published, then reruns from scratch (`tool.ts:99-105`). Otherwise settles `failed` with error result "Tool ${name} was interrupted and may have partially run" (`tool.ts:106-110`); `failed` records cancellation intent so the call's owned conversations are aborted. README: "the model gets an `interrupted` error result with the output committed so far" (`packages/durable/README.md:168`).
  - Both-safe rule protects against a hot-reloaded definition that downgraded safety and against old intents written under an older safe definition.
- **Defaults**: none of the built-in durable coding tools (`packages/durable/src/tools/*`) declares `replay` (grep → no hits) → read/write/edit/bash/powershell all `unsafe`.
- **Replay-safe examples**:
  - Experimental durable TUI `subagent` tool: `replay: "safe"` — "A rerun after a crash finds the child it created and the submission it made": `scanConversations({ownerTaskId})` reuse + `requestId: subagent:${taskId}` idempotence; child removes the Subagent extension so it cannot recurse (`packages/coding-agent/src/experimental/durable/subagent.ts:30-45`) — [[task-owned-subagent]].
  - Vacation demo: parallel canned `search` tools (weather 6 s, museums 10 s, trains 30 s) `replay:"safe"`; crash mid-run + `--continue` keeps finished searches and reruns only the unfinished one (`packages/coding-agent/src/experimental/vacation/vacation.ts:16-55`; `vacation/README.md:32-34`; `395315f48`).
- **Related task-level guarantees** (`spec.md`): missing/incompatible task code never terminalizes a task — stays `pending` and `blocked: missing_task | task_too_old | migration_failed` (`spec.md:2011-2032`; `harness/scheduler.ts:57`); only abort orphans it, and `orphaned` means "no task code ran, so external effects the task started may remain uncleaned" (`spec.md:2034-2039`). Hot reload hands a task over at the next phase boundary (`spec.md:2077-2087`). Abort protocol: commit abortRequested → signal and join → wait for owned work → abort handler commits terminal (`spec.md:1935-1941`).
- Tool output is committed progressively (`pi.live`, progress throttles 100 ms / 100 KiB/s) so an interrupted result still carries output so far (`harness/tool.ts:294-327`; `harness/output.ts:261`).
- MCP analogue in stable pi: MCP tool calls are never retried because the "server may already have performed them" (`docs/mcp.md:248`) — [[mcp-integration]].

## Constants
| name | value | path:line |
|---|---|---|
| default replay policy | `"unsafe"` | `packages/durable/src/harness/tool.ts:88` |
| interrupted message | "Tool ${name} was interrupted and may have partially run" | `harness/tool.ts:106` |
| `PROGRESS_BYTES_PER_SECOND` | 100 KiB/s | `packages/durable/src/harness/output.ts:261` |

## Evolution
- 2026-05-03 `a5b27367d` — first `AgentHarness` in `packages/agent/src/harness`.
- 2026-08-11 `5aeb06bb1` "establish durable harness type contracts" — `replay?: "never" | "safe"` added to stable `AgentTool` (`packages/agent/src/types.ts:489-490`, "Recovery policy for an effect whose durable intent exists but whose outcome is unknown").
- 2026-09-07 → 09-17 Pico spec redesign (`73f3257dd` … `729d5cb74`); 2026-09-18 `080160162` `packages/durable`.
- 2026-09-29 `445770e03` (Package 16) first tool turn with intent commit; `2532a0bef` (Package 18) ownership + subagent handles.
- 2026-09-30 `37c9d0d20` "harden the persistent subagent example": background subagent no longer offers wait and "is not rerun after a crash (a repeated stop could stop newer work)".
- 2026-10-01 `395315f48` vacation demo with replay-safe parallel searches.

## Evidence commits
`a5b27367d`, `5aeb06bb1`, `080160162`, `445770e03`, `2532a0bef`, `37c9d0d20`, `395315f48`.

## Quirks
- Even `read` (side-effect-free) is `unsafe` by default → a crash mid-read yields an "interrupted" error rather than a silent rerun; deliberate? (open question, 10-durable findings) `(unverified intent)`.
- **Vocabulary mismatch**: stable `packages/agent/src/types.ts:490` declares `replay?: "never" | "safe"` (from `5aeb06bb1`), while durable uses `"safe" | "unsafe"`; the stable agent-loop never reads `replay` (grep of `packages/agent/src` finds no other use) — vestigial contract.
- Replay safety is per-operation, not per-tool-type: a stop/kill is not idempotent with respect to newer work (`37c9d0d20` lesson).
- Spec footgun: a subagent tool must not report child usage or subtree sums double count (`spec.md:4634-4636`).

## Failures
- [[non-idempotent-tool-replayed-after-crash]]
