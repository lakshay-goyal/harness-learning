---
type: implementation
harness: pi
concept: run-settlement
commit: b30a6dd77
files: [packages/coding-agent/src/core/agent-session.ts:1821, packages/coding-agent/src/core/agent-session.ts:1852, packages/coding-agent/src/core/agent-session.ts:1892, packages/coding-agent/src/core/agent-session.ts:1078, packages/coding-agent/src/core/agent-session.ts:1197, packages/agent/src/agent.ts:350, packages/agent/src/agent.ts:565]
---
[[run-settlement]] in [[pi]].

## Mechanism
- **Two boundaries**: `agent_end` closes ONE low-level `runLoop` run; `agent_settled` = "Pi will not continue automatically" (`packages/coding-agent/docs/json.md:48,56`; `docs/extensions.md:66-67`). Added `e9fa5a68a` 2026-07-09 (#6363).
- **Driver loop** `_runAgentPrompt` (`agent-session.ts:1821-1850`): reset `_agentRunAbortRequested`, clear `_failedResponse` ("the new prompt replaces" a scheduled retry, `:1823-1824`), `await agent.prompt(msgs)`, then `while (!_agentRunAbortRequested)`: `_handlePostAgentRun()` → true ⇒ `agent.continue()`; else `_runBeforeSettleBoundary()` → true ⇒ `agent.continue()`; else break. `finally`: `_finishCancelledRetry` if aborted, clear `_failedResponse` + per-run prompt options, flush pending bash + custom messages ([[out-of-band-message-deferral]]), `await _emitAgentSettled()` (`:1842-1849`).
- `_handlePostAgentRun` order (`agent-session.ts:1852-1890`): abort → `_finishCancelledRetry`, false (`:1857-1860`); no message → `hasQueuedMessages()` (`:1861`); retryable + `_prepareRetry` → set `_failedResponse`, continue (`:1863-1867`) → [[pi--auto-retry-backoff|auto-retry-backoff]]; error after retries → `auto_retry_end{success:false}` (`:1873-1881`); `_checkCompaction(msg, skipAborted=true, toolResults)` (`:1883`) → [[overflow-recovery]], [[auto-compaction]]; finally continue iff `agent.hasQueuedMessages()` — "Messages queued by agent_end handlers require a fresh run before pre-settlement handlers fire" (`:1887-1889`, `a29a7902e`).
- `_runBeforeSettleBoundary` (`agent-session.ts:1892-1914`): extension `agent_before_settle{outcome}` may append draft entries and request **one** continuation; no handlers ⇒ `hasQueuedMessages()`; abort during boundary ⇒ false; continuation rejected + reported if context can't continue (last LLM role assistant and nothing queued) (`:1906-1908`; `:999-1036`) → [[extension-event-hooks]].
- `_emitAgentSettled` (`agent-session.ts:1078-1097`): notifies cache warmer `onAgentSettled()` ([[cache-warming]]), marks run inactive, awaits extension `agent_settled` then public emit, then runs `_deferredSettledActions` (re-entrant `prompt()`/`sendCustomMessage(triggerTurn)` called during settled emission are deferred, `:1968-1971`, `2314-2317`, `1089-1097`), then resolves idle waiters.
- `agent_end.willRetry` (session event, `agent-session.ts:198-202`, `:1140`) computed synchronously by `_willRetryAfterAgentEnd` (`:1197-1211`): false if abort requested / retry disabled / attempts exhausted, else `_isRetryableError(last assistant)`.
- **Idle** = no active run AND not compacting (`agent-session.ts:1464-1466`); compaction + branch summary included in idle tracking (`bea67d90d`, #8920). `Agent.waitForIdle()` (`agent.ts:350`).
- **Awaited listeners**: `Agent.subscribe()` listeners are awaited, get the abort signal; `waitForIdle/prompt/continue` settle only after awaited `agent_end` listeners (`9022a5b5e`; `agent.ts:256-269`, `565-612`). Reducer updates state BEFORE listeners (`agent.ts:565-603`; `2b0aa5ed8`). Session `_emit` to public listeners synchronous/not awaited (`agent-session.ts:1039-1043`); extension emits awaited; extensions see each event before public listeners (`:1138-1140`). Low-level `agentLoop()` streams are observational — no barrier (`packages/agent/README.md:560`).
- `message_end` extension handlers may return a replacement message, applied by **in-place mutation** so agent state, later events and persistence agree (`agent-session.ts:1266-1280,1314-1332`).
- Pre-prompt compaction check (`prompt()`, `agent-session.ts:2051-2056`) no longer calls `continue()` (`73581ea99`) — would have replayed before the new prompt.
- Session replacement (`/new`, `/resume`, `/fork`) settles first: `session.abort()` "so the aborted turn (including tool results) is persisted to the outgoing session" (`agent-session-runtime.ts:167-178`; `cefa40ed8`).

## Constants
| name | value | path:line |
|---|---|---|
| before-settle continuations per boundary | 1 | `agent-session.ts:1892-1914` |
| RPC client wait defaults (`waitForIdle/collectEvents/promptAndWait`) | 60000 ms | `packages/coding-agent/src/modes/rpc/rpc-client.ts:471,491,513` |

## Evolution
- Originally fire-and-forget event handlers.
- 2026-02-06 `b050c582a` resume queued msgs after auto-compaction via `setTimeout(() => agent.continue(), 100)` (#1312).
- 2026-03-02 `dfc779faa` serialized handling via promise chain `_agentEventQueue` (tool results persisted before their assistant msg, #1717).
- 2026-03-02 `890329907` retry promise created synchronously at `agent_end` dispatch (#1726).
- 2026-03-20 `8a0529ed9` `_resolveRetry()` moved from `message_end` to `agent_end` — a retry that produced tool calls let `prompt()` return while tools ran (#2440).
- 2026-03-30 `9022a5b5e` awaited subscribers; `agent_end` no longer the idle boundary.
- 2026-05-19 `32bcdc973` + `c685b2736` deleted `_agentEventQueue` + `_retryPromise`, replaced by synchronous driver loop `_runAgentPrompt`/`_handlePostAgentRun`; `agent_end.willRetry`.
- 2026-05-28 `a29a7902e` (PR #5115 merge `8e77f8797`) drain follow-ups queued during `agent_end`.
- 2026-06-25 `73581ea99` pre-prompt compaction stops calling `continue()`.
- 2026-07-09 `e9fa5a68a` `agent_settled` event (#6363).
- 2026-09-03 `bea67d90d` idle includes compaction.

## Evidence commits
`b050c582a` `dfc779faa` `890329907` `8a0529ed9` `9022a5b5e` `32bcdc973` `c685b2736` `a29a7902e` `8e77f8797` `73581ea99` `e9fa5a68a` `bea67d90d` `2b0aa5ed8` `cefa40ed8`

## Quirks
- `agent_end` fires possibly several times per user prompt (retries, compaction-retry, queued work); consumers wanting "done" must use `agent_settled` / `waitForIdle` (`docs/json.md:48,56`).
- `_runAutoCompaction` uses its own controller; whether `prepareNextTurn` compaction can start in the window after an abort is open (unverified).

## Durable variant (packages/durable)
- "Busy" = presence of `pi.live.run` (`packages/durable/docs/spec.md:2208-2217`); run baton handed generation→next generation; `taskId` names the generation that settles the inputs.
- Admission/terminal table: run answers → input `done` with answer entry; run fails/aborts → input `unanswered` + reason, inbox untouched (`spec.md:2219-2232`). Only successful ends (answer, `terminate`, `handoff`) apply the `final` boundary (`spec.md:2298-2302`).
- `onYield` hooks may continue a `stop`/`length` answer before the final boundary (`spec.md:3536-3566`) ≈ `agent_before_settle`.
- Idle waits respect ownership: parent idle only once owned child work is idle; background tasks are idle boundaries (`spec.md:1958-1965`, `packages/durable/README.md:452-454`) → [[pi--task-owned-subagent|task-owned-subagent]].

## Failures
[[retry-wait-race-prompt-returns-early]] · [[queued-messages-stranded-at-run-end]] · [[listeners-see-stale-agent-state]] · [[compaction-cancellation-races]]
