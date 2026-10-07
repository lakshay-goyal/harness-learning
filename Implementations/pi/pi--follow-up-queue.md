---
type: implementation
harness: pi
concept: follow-up-queue
commit: b30a6dd77
files: [packages/agent/src/agent.ts:304, packages/agent/src/agent-loop.ts:301, packages/agent/src/agent.ts:384, packages/coding-agent/src/core/agent-session.ts:1861, packages/coding-agent/src/core/agent-session.ts:2088]
---
[[follow-up-queue]] in [[pi]].

## Mechanism
- `followUp(msg)` — "Queue a message to run only after the agent would otherwise stop" (`packages/agent/src/agent.ts:303-306`). Second `PendingMessageQueue` (`agent.ts:143-174`, `:248`), mode `all | one-at-a-time`, default one-at-a-time (`settings-manager.ts:864`).
- Polled only at the natural-stop point: no tool calls, no steering → `getFollowUpMessages()`; non-empty ⇒ set as pending, `continue` outer loop (`agent-loop.ts:301-308`; contract `types.ts:295-306`).
- `continue()` from an assistant tail: steering batch first, then one follow-up batch (`agent.ts:392-405`); a non-assistant tail continues, follow-ups wait for natural stop (`packages/agent/README.md:171`).
- Session layer re-checks after the low-level loop: `_handlePostAgentRun` returns `agent.hasQueuedMessages()` when nothing else to do — catches follow-ups queued by `agent_end` handlers (`agent-session.ts:1861`, `1887-1889`; `a29a7902e`); before-settle boundary also continues if queued (`:1893`, `:1905`) → [[pi--run-settlement|run-settlement]].
- AgentSession `prompt(..., {streamingBehavior:"followUp"})` routes to `agent.followUp` (`agent-session.ts:2018`). `sendCustomMessage` matrix: `deliverAs:"nextTurn"` → stored, attached to next user prompt (`:2088-2091`); streaming + triggerTurn → steer/followUp; idle + triggerTurn → starts a run (`:2292-2328`).
- Durable background subagent reports arrive as follow-ups: vacation `report` phase submits `{whenBusy:"followUp", requestId:"research-report:<task>"}` to main (`packages/coding-agent/src/experimental/vacation/vacation.ts:94-98`) → [[pi--task-owned-subagent|task-owned-subagent]].

## Constants
| name | value | path:line |
|---|---|---|
| `followUpMode` default | `"one-at-a-time"` | `packages/coding-agent/src/core/settings-manager.ts:864`; `packages/agent/src/agent.ts:248` |

## Evolution
- 2026-01-02 `d0a4c3702` split from single queue (#403).
- 2026-02-06 `b050c582a` queued msgs resumed after auto-compaction (#1312) — originally `setTimeout(100)` continue.
- 2026-05-28 `a29a7902e` / `8e77f8797` (#5115) drain follow-ups queued during `agent_end`.
- 2026-05-19 `32bcdc973` driver loop replaces timer-based continuation.

## Evidence commits
`d0a4c3702` `b050c582a` `a29a7902e` `8e77f8797` `32bcdc973`

## Quirks
- Low-level loop drains both queues before `agent_end`; anything enqueued by `agent_end` handlers needs a fresh run (`agent-session.ts:1887-1888` comment).

## Durable variant (packages/durable)
- `whenBusy` default = followUp (`packages/durable/README.md:311-321`); placed only at the `final` boundary of a **successful** run end, starting one successor generation (`spec.md:2253-2256`, `2287-2296`). Failed run keeps the follow-ups queued until the next submission (`packages/durable/README.md:323`).

## Failures
[[queued-messages-stranded-at-run-end]] · [[side-phase-input-lost]]
