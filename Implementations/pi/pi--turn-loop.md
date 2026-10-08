---
type: implementation
harness: pi
concept: turn-loop
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:163, packages/agent/src/agent-loop.ts:381, packages/agent/src/agent-loop.ts:508, packages/agent/src/agent.ts:372, packages/agent/src/agent.ts:507, packages/coding-agent/src/core/agent-session.ts:1821, packages/durable/src/harness/generation.ts:420]
---
[[turn-loop]] in [[pi]].

## Mechanism
- **Three nested loops.** L1 inner turn loop + L2 outer follow-up loop live in stateless `runLoop` (`packages/agent/src/agent-loop.ts:163-321`); L3 = session post-run driver `_runAgentPrompt` in `AgentSession` (`packages/coding-agent/src/core/agent-session.ts:1821-1850`) → see [[pi--run-settlement|run-settlement]].
- Entry points: `agentLoop`/`runAgentLoop` (new prompt msgs; emits `agent_start`, `turn_start`, `message_start/end` per prompt msg before `runLoop`, `agent-loop.ts:38-61,102-126,117-122`) and `agentLoopContinue`/`runAgentLoopContinue` (no new message; throws if context empty or tail is `assistant`, `agent-loop.ts:71-100,128-151`).
- Initial steering poll before first request ("user may have typed while waiting") (`agent-loop.ts:175-176`).
- Outer `while (true)` (`agent-loop.ts:179`) exists only for follow-ups (`:301-308`) or one explicit `finishTurn` continue (`:310-314`).
- Inner `while (hasMoreToolCalls || pendingMessages.length > 0)` (`agent-loop.ts:183`). One iteration = one turn = "one assistant response + any tool calls/results" (`packages/agent/src/types.ts:520`). Steps:
  1. Not first turn → `prepareNextTurn(lastCompletedTurn)` may swap context/model/thinking + append msgs (`:185-200`); re-poll steering only if earlier poll empty (`:201-206`); emit `turn_start` (`:207`).
  2. `declareToolChanges` inserts tool-delta system msg; prepared + pending msgs appended with `message_start/end` (`:210-217`) → [[transcript-carried-system-prompt]].
  3. `prepareRequest` before every provider request incl. first (`:219-239`).
  4. `streamAssistantResponse` (`:242`): `transformContext` → `convertToLlm` → `normalizeContext` → per-request `getApiKey` → `streamFn` (`:381-407`) → [[message-conversion-layer]], [[context-transform-hook]].
  5. **Hard exit** on `stopReason` `error`/`aborted`: `finishTurn` called (decision ignored), `turn_end`, `agent_end`, return (`:245-256`).
  6. Tool calls: `stopReason === "length"` → `failToolCallsFromTruncatedMessage` (`:263-270`, `:478-503`) → [[pi--truncated-tool-call-guard|truncated-tool-call-guard]]; else `executeToolCalls` (`:508`). `hasMoreToolCalls = !batch.terminate` (`:272`).
  7. `finishTurn` before `turn_end`; `{action:"end"}` → `agent_end` + return without polling queues (`:280-292`).
  8. Steering poll after the turn (`:295`); explicit continuation cleared if tools/steering already cause a request (`:294-298`).
- Loop continues purely on **presence of tool calls** (not on `stopReason === "toolUse"`) — so providers must not promote a length/error stop to `toolUse` (`5093641a5`, see [[length-truncated-tool-calls-executed]]).
- **Batch early termination**: `shouldTerminateToolBatch` = non-empty batch AND every finalized result `terminate === true` (`agent-loop.ts:690-692`); mixed batch continues (`packages/agent/README.md:120`); steering/follow-up still polled after (`:295-308`); `terminate` is runtime-only, persisted toolResult is ordinary (README "Error Handling"). Blocked `beforeToolCall` can set `terminate` (`1eb988cfe`, #7715) → [[structured-tool-output]].
- Tool execution inside the turn: default `toolExecution: "parallel"` (`agent.ts:253`); any call to a tool with `executionMode:"sequential"` makes the whole batch sequential (`agent-loop.ts:515-521`); parallel = sequential preflight → `Promise.all` → `tool_execution_end` in completion order, toolResult messages in assistant source order (`agent-loop.ts:586-660`, `759d55152` #3503) → [[parallel-tool-execution]]. Every failure → `isError` result (`agent-loop.ts:845-853`, `802-810`) → [[tool-error-as-result]].
- **Barrier**: `Agent.processEvents` awaits listeners, so assistant `message_end` processing (incl. session persistence) completes before tool preflight; `beforeToolCall` sees the assistant msg in state (`agent.ts:565-612`; `packages/agent/README.md`).
- Thrown exception from hooks/convertToLlm/stream → `handleRunFailure` synthesizes empty assistant msg `stopReason` `aborted|error` + `errorMessage`, emits `message_start/end`, `turn_end`, `agent_end` so the session listener persists it (`agent.ts:523-548`).
- **Non-reentrant**: `prompt()` while active throws "Agent is already processing a prompt. Use steer() or followUp()…" (`agent.ts:372-380`); `continue()` likewise (`:384-387`); `runWithLifecycle` guard (`:507-510`); `reset()` refuses during run (`1532c9994`, #7717) → [[reentrant-prompt-corrupts-state]].
- `agent_end` carries `newMessages` (prompt run includes prompt msgs; continuation excludes pre-existing context) (`types.ts:142`, `agent-loop.ts:320`).
- Termination table:

| Condition | Where |
|---|---|
| `stopReason` error/aborted | `agent-loop.ts:245-256` (finishTurn decision ignored, `types.ts:150-153`) |
| `finishTurn` → end | `agent-loop.ts:289-292` |
| no tool calls, no steering, no follow-up, no explicit continuation | `agent-loop.ts:316-317` |
| all results `terminate:true` | `agent-loop.ts:272`, `690-692` |
| thrown exception | `agent.ts:523-548` |
| session abort requested | `agent-session.ts:1832-1839` |

- **No max-turn cap** anywhere in agent-core / core session (no `maxTurns|maxIterations|maxSteps`); README warns unconditional `finishTurn` continue = endless loop (`packages/agent/README.md:146`) → [[no-turn-cap]]. Bounded recovery instead (retry max 3, overflow compact-once).
- Default stream fn is a global registry set by coding-agent (`packages/agent/src/stream-fn.ts:3-20`; `sdk.ts:44-46`) so agent-core does not depend on all providers (`1235c0ec6`, `b9e5c5d94` #6915). `streamProxy` strips `partial` from deltas server-side, client rebuilds (`proxy.ts:20-46,72-77`).

## Constants
| name | value | path:line |
|---|---|---|
| max turns | none (absent) | `packages/agent/src/agent-loop.ts` |
| `toolExecution` default | `"parallel"` | `packages/agent/src/agent.ts:253` |
| `transport` default | `"auto"` | `packages/agent/src/agent.ts:251` |
| `DEFAULT_THINKING_LEVEL` | `"medium"` | `packages/coding-agent/src/core/defaults.ts:3` |
| durable `DEFAULT_POLL_AFTER_MS` | 5000 | `packages/durable/src/harness/generation.ts:109` |

## Evolution
- 2025-12-09 `2b0aa5ed8` update agent state **before** emitting events (handlers saw stale state).
- 2025-12-19 `1167e8445` `getApiKey` per LLM call — Copilot OAuth (~30 min) expired during long tool phases (#223).
- 2025-12-20 `117af076c` steering interrupts tool batch → reverted 2026-03-16 `208a2cc12` (see [[pi--steering-queue|steering-queue]]).
- 2026-01-02 `5ef3cc90d` guard concurrent `prompt()`.
- 2026-03-14 `63ac2df24` `beforeToolCall`/`afterToolCall` hooks added + `toolExecution: "parallel"|"sequential"` (parallel default, source-ordered results) (#2113).
- 2026-03-30 `9022a5b5e` listeners awaited → message_end barrier.
- 2026-04-16 `b9cd557d1` `afterToolCall` throw → error result instead of aborting parallel batch (#3084).
- 2026-04-22 `759d55152` eager `tool_execution_end`, source-ordered persisted results (#3503).
- 2026-05-09 `322759a3f` turn-state snapshots (`prepareNextTurn` origin) → [[pi--turn-lifecycle-hooks|turn-lifecycle-hooks]].
- 2026-06-12 `daab056ac` ignore tool progress after settle (`acceptingUpdates`, `agent-loop.ts:826-857`, #5573).
- 2026-07-07 `351efc828` length-stopped tool calls failed (#6285).
- 2026-09-21 `466db0fec` canonical projection installed via `prepareRequest`; `finishTurn` added.
- 2026-10-01 `7fd478a2e` removed earlier in-agent `AgentHarness` (105k lines) — agent-core = Agent + loop + proxy only (`packages/agent/CHANGELOG.md:25`).

## Evidence commits
`2b0aa5ed8` `1167e8445` `5ef3cc90d` `63ac2df24` `9022a5b5e` `b9cd557d1` `759d55152` `322759a3f` `daab056ac` `351efc828` `1eb988cfe` `466db0fec` `1532c9994` `7fd478a2e` `1235c0ec6` `b9e5c5d94`

## Quirks
- One slow parallel tool delays all toolResult messages and the next request (source-order emission, `agent-loop.ts:646-654`) (inferred).
- After abort during tool execution there is no signal check between tool batch and next request; loop proceeds to `prepareNextTurn`/`prepareRequest`/`streamFn`, provider fails immediately as `aborted` → one extra aborted assistant persisted (inferred from `agent-loop.ts:183-242`; no test found) (unverified).
- Hook/listener exceptions: extension `tool_call` error re-thrown → blocked/error result (fail-closed) (`agent-session.ts:658-681`) → [[tool-call-gate]].
- `deferred` stopReason is non-error/non-tool → natural stop in stable loop (inferred); durable handles it as `poll` phase.

## Durable variant (packages/durable)
- `while` loop replaced by a **chain of durable tasks** handing off the `pi.live.run` baton: `pi.generation` → `pi.tool`×n (waited `allSettled`) → next `pi.generation` (`packages/durable/docs/spec.md:2208-2217,3576-3594,3651-3673`) → [[durable-execution]].
- Generation phases `prepare`/`request`/`retry`/`poll`/`tools` (`generation.ts:53,61,71,73,83`), start `{phase:"prepare",attempt:1}` (`:118`).
- Classification (`generation.ts:420-500`): `throwIfAborted` first (`:426`); `deferred` → `poll` with `pollAfterMs ?? 5000` (`:429-448`); `toolUse` **with** calls → tool round (`:452-453`); `stop`/`length`/`toolUse` → `answer` (`:455-456`) — a length-stopped message's tool calls are never run; overflow error → owned compaction once (`:460-476`); retryable → `retry` phase (`:477-497`); else run ends `unanswered/model_error` (`:499-500`).
- Round terminate requires **every** call incl. those answered without a task (`tool_unavailable`) (`generation.ts:613-616`); last `handoff` in call order wins (`:618-628`) → [[session-handoff]].
- Tool rounds parallel by default (`harness/agent.ts:68`); sequential if any `executionMode:"sequential"`; call to a tool not offered → `tool_unavailable` result, no task (`spec.md:3576-3594`).

## Failures
[[reentrant-prompt-corrupts-state]] · [[late-tool-progress-after-settlement]] · [[listeners-see-stale-agent-state]] · [[length-truncated-tool-calls-executed]] · [[steering-skips-pending-tool-calls]] · [[tool-preflight-ignores-abort]] · [[proxied-stream-option-loss]]
