---
type: implementation
harness: pi
concept: agent-event-stream
commit: b30a6dd77
files: [packages/agent/src/types.ts:516-539, packages/agent/src/agent-loop.ts:423-431, packages/coding-agent/src/core/agent-session.ts:186-237, packages/coding-agent/src/core/agent-session.ts:1138-1140, packages/coding-agent/src/modes/json-event.ts:40-61, packages/coding-agent/src/modes/print-mode.ts:109-127, packages/coding-agent/docs/json.md:5-190]
---
[[agent-event-stream]] in [[pi]].

## Mechanism
- Core `AgentEvent` (`packages/agent/src/types.ts:516-539`): `agent_start`, `agent_end{messages}`, `turn_start`, `turn_end{message, toolResults}`, `message_start`, `message_update{message, assistantMessageEvent}` (assistant streaming only), `message_end`, `tool_execution_start{toolCallId,toolName,args}`, `tool_execution_update{partialResult}`, `tool_execution_end{result,isError,durationMs?}`.
- Provider-level `assistantMessageEvent` forwarded inside `message_update`: text/thinking/toolcall `_start/_delta/_end` (`agent-loop.ts:423-431`); `message_update` carries a shallow copy `{...partialMessage}` → [[unified-provider-api]].
- `AgentSessionEvent` (`coding-agent/src/core/agent-session.ts:196-237`) adds `agent_end{willRetry}`, `agent_settled`, `queue_update{steering,followUp}`, `compaction_start/end{reason: manual|threshold|overflow, willRetry, aborted, errorMessage}`, `entry_appended`, `session_info_changed`, `thinking_level_changed`, `auto_retry_start{attempt,maxAttempts,delayMs,errorMessage}`, `auto_retry_end{success,attempt,finalError}`, `summarization_retry_*`, `bash_execution_update`; tool events gain `parentToolCallId` for nested calls (`agent-session.ts:186-193`).
- Ordering: extensions get each event before public listeners (`agent-session.ts:1138-1140`); `message_end` plugin replacement applied by mutating the object in place so agent state, later events and persistence agree (`agent-session.ts:1266-1280,1314-1332`). Agent state reducer updates BEFORE listeners (`packages/agent/src/agent.ts:565-603` `processEvents`; `2b0aa5ed8`). Public `_emit` synchronous (`agent-session.ts:1039-1043`); extension emits awaited.
- `agent_end` = one low-level run; `agent_settled` = "Pi will not continue automatically" (`docs/json.md:48,56`; `docs/sdk.md:72` "Subscribing to events") → [[run-settlement]].
- **JSON mode** (`--mode json`): one session header record, then `AgentSessionEvent` JSONL (`docs/json.md:5-9,21-27`; `modes/print-mode.ts:109-127`). `message_update` stripped of cumulative `partial`/`message` snapshots → deltas + constant-size usage only (`modes/json-event.ts:40-61`; `a4475344f`, #7290). Extra types documented: `queue_update`, `entry_appended`, `session_info_changed`, `thinking_level_changed`, `compaction_start/end`, `auto_retry_start/end`, `summarization_retry_*` (`docs/json.md:117-180`).
- Consumers: TUI (`InteractiveMode`), SDK `session.subscribe` ([[pi--sdk-embedding]]), JSON mode, RPC ([[pi--headless-rpc-mode]]), extensions ([[pi--extension-event-hooks]]), subagent example (`--mode json -p --no-session`, [[subagent-as-subprocess]]).

## Constants
| name | value | path:line |
|---|---|---|
| core event types | 10 | `packages/agent/src/types.ts:516-539` |
| durable watch frame cap before snapshot | 100 | `packages/durable/src/session/observation.ts:15` |

## Evolution
- 2026-02-12 `ff5148e7c` message/tool_execution events forwarded to extensions (#1375).
- 2026-03-02 `dfc779faa` events serialized via `_agentEventQueue` (tool results persisted before tool call, #1717) → 2026-03-30 `9022a5b5e` awaited `Agent.subscribe()` listeners → 2026-05-19 `32bcdc973`/`c685b2736` synchronous post-run driver ([[run-settlement]]).
- 2026-07-09 `e9fa5a68a` `agent_settled` (#6363).
- 2026-08-03 `a4475344f` linear JSON streaming output (#7290 quadratic).

## Evidence commits
`ff5148e7c` `dfc779faa` `9022a5b5e` `32bcdc973` `e9fa5a68a` `a4475344f` `2b0aa5ed8`

## Quirks
- In-process `message_update` still carries the partial message (cheap by reference) — only the wire strips it.
- Plugins can delay every downstream consumer since their handlers are awaited first.

## Durable variant (packages/durable)
- UIs render only committed state: `viewState()`/`watch()` stream exact Chord ops per commit; late joiners start from the current view (`packages/durable/README.md:276-302`). `watchEvents()` derives coding-agent-style events from commits; consumer >100 batches behind gets a fresh snapshot (`packages/durable/README.md:368-382`, cap stated at `:382`; `packages/durable/src/harness/events.ts:137-156`; cap enforced by `CommittedWatch` `packages/durable/src/session/observation.ts:15,224`) → [[replicated-state]].

## Failures
[[quadratic-event-stream-output]] · [[headless-protocol-stream-corruption]]
