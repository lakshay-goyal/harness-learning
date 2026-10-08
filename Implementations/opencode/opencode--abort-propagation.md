---
type: implementation
harness: opencode
concept: abort-propagation
commit: ecc4916b5a
files: [packages/opencode/src/session/run-state.ts:77-145, packages/opencode/src/effect/runner.ts:171-213, packages/opencode/src/session/processor.ts:553-611, packages/opencode/src/session/processor.ts:661-669, packages/opencode/src/session/llm.ts:357-366, packages/opencode/src/session/tools.ts:59-64, packages/opencode/src/session/message-v2.ts:338-373, packages/opencode/src/tool/shell.ts:533-567, packages/core/src/session/run-coordinator.ts:94-101, packages/core/src/session/runner/llm.ts:286-320]
---
[[abort-propagation]] in [[opencode]].

## Mechanism

### Legacy runtime — fiber interruption bridged to one `AbortController`
1. `POST /session/:id/abort` → `SessionPrompt.cancel` → `SessionRunState.cancel` (`packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:232-234`).
2. `SessionRunState.cancel` first cancels background jobs **transitively** (jobs whose id / `metadata.sessionId` / `metadata.parentSessionId` match, iterated to closure = subagent sessions), then the runner (`packages/opencode/src/session/run-state.ts:77-86,111-145`).
3. `Runner.cancel`: `Fiber.interrupt(run.fiber)`, fail the `done` Deferred with `Cancelled`; shell state waits for the shell `ready` latch first (`packages/opencode/src/effect/runner.ts:171-213`).
4. Processor `Effect.onInterrupt` → `halt(new DOMException("Aborted","AbortError"))` → `AbortedError` on the assistant message (`packages/opencode/src/session/processor.ts:661-669`).
5. `LLM.stream` holds an `acquireRelease` `AbortController`; scope close → `ctrl.abort()` → AI SDK `abortSignal` → fetch aborted (`packages/opencode/src/session/llm.ts:357-366`). The same signal reaches every tool as `Tool.Context.abort` (`packages/opencode/src/session/tools.ts:59-64`).
6. Tools: shell races exit/abort/timeout and kills with `forceKillAfter: "3 seconds"`, appending "User aborted the command" (`packages/opencode/src/tool/shell.ts:533-567`); task tool cancels the child session (`packages/opencode/src/tool/task.ts:296-325`).
7. `cleanup()` (ensuring): close open text/reasoning parts, wait ≤250 ms per pending tool, mark still-running tools `status:"error"`, `error:"Tool execution aborted"`, `metadata.interrupted:true` (keeping streamed metadata), set `time.completed` (`processor.ts:553-611`).
8. **Replay**: an interrupted tool whose `metadata.output` is a string (shell streams it) is replayed as a **successful** `output-available` result with the partial output; other errors as `output-error`; still `pending|running` → `"[Tool execution was interrupted]"` (`packages/opencode/src/session/message-v2.ts:338-373`).

### v2 runtime — Effect interruption only
- `POST /api/session/:sessionID/interrupt` → `V2Session.interrupt` wrapped `Effect.uninterruptible` (`packages/core/src/session.ts:430-431`) → coordinator marks `stopping`, clears `pendingWake`, `Fiber.interrupt(owner)` (`packages/core/src/session/run-coordinator.ts:94-101`).
- No AbortController plumbing: "Effect interruption is the cancellation mechanism. Tools … must not translate interruption or defects into model-visible failures" (`specs/v2/tools.md:52`).
- Settlement region: `FiberSet.clear(toolFibers)`, unsettled tools → `Tool.Failed "Tool execution interrupted"`, active assistant → "Provider turn interrupted" (`packages/core/src/session/runner/llm.ts:304-320`). Bash children spawned detached with `forceKillAfter` 3 s (`packages/core/src/tool/bash.ts:160-166`).
- Durable inbox rows are kept; no "interrupted" status event (`session.next.interrupt.requested.1` removed, `specs/v2/schema-changelog.md:23-26`).

## Constants
| name | value | path:line |
|---|---|---|
| pending-tool settle wait on cleanup | 250 ms | `packages/opencode/src/session/processor.ts:587` |
| shell `forceKillAfter` | 3 s | `packages/opencode/src/tool/shell.ts:550,554`; `packages/opencode/src/session/prompt.ts:564` |
| `SIGKILL_TIMEOUT_MS` (core shell helper) | 200 | `packages/core/src/shell.ts:12` |

## Evolution
- 2025-10-16 `fc18fc8a08` bash hangs & orphans (kill escalation).
- 2026-01-29 `e5b33f8a5e` `AbortSignal` in `Ripgrep.files()`/glob; 2026-02-03 `93e060272a` `abortAfterAny` clears timers (leaked `AbortSignal.any` closures).
- 2026-04-08 `2bdd279467` abort reaches the inline read for @-file mentions.
- 2026-04-09 `c29392d085` interrupted bash output preserved; `3199383eef` finalized via the normal tool-result path.
- 2026-04-27 `139c4fd555` shell cancel during startup (`ready` latch); 2026-06-03 `2a33addd29` shell cancel race.
- 2026-05-04 `75d141b574` subtask child sessions cancelled with the parent.
- 2026-05-12 `822eec0d62` cancel failed `done` instead of awaiting the run.
- 2026-06-05 `12e38866ed` v2 interrupt; 2026-06-08 `79cff288a6` abort signal passed to MCP tool calls.

## Quirks / drift
- **Partial shell output replayed as success**: after Esc the model sees an `output-available` result, not an error — it may treat a killed command as finished (design choice of `c29392d085`; no follow-up fix).
- Title/summary/prune side calls are forked into the layer scope and ignore user abort ([[opencode--auxiliary-model-calls|auxiliary-model-calls]]).
- Two cancellation models coexist: AbortSignal (legacy, AI SDK) vs pure fiber interruption (v2).

Failures: [[tool-preflight-ignores-abort]] · [[ownership-cancellation-races]] · [[interrupted-message-never-finalized]] · [[orphaned-tool-calls-and-results]] · [[session-switch-leaves-dangling-tool-calls]].

Contrast: [[pi--abort-propagation|pi]] persists the aborted partial and skips it on replay; opencode legacy replays interrupted shell output as a successful tool result.
