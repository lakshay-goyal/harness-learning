---
type: failure
concepts: [run-settlement, agent-event-stream, abort-propagation]
harnesses: [pi, codex]
---
**Symptom** — Event handlers on `message_end` (and other events) read agent state that did not yet include the event's effect.

**Root cause** — Events were emitted before the agent state reducer applied them.

**Fix · [[pi]]** — `2b0aa5ed8` 2025-12-09 "update state before emitting events": reducer runs first, then listeners (`packages/agent/src/agent.ts:565-603`). Later `9022a5b5e` 2026-03-30 made listeners awaited so assistant `message_end` processing is a barrier before tool preflight (`agent.ts:565-612`).

**Fix · [[codex]]**
- Symptom: "`TurnAborted` could be emitted before extension abort callbacks finished, allowing consumers to observe a terminal event while lifecycle cleanup was still pending"; clients that synchronously re-read the rollout on `TurnAborted` didn't see the interrupt marker.
- `9ef9cb1d9f` 2026-09-30 (#49475): await extension `on_turn_abort` and flush the rollout before sending `TurnAborted`, flush again after (`codex-rs/core/src/tasks/mod.rs:982-1028`); normal completion flushes before `on_task_finished` (`:357-374`).

**Lesson** — Reduce state, then notify; terminal events are a barrier — emit them only after listeners, callbacks and persistence that later phases depend on are complete.

Related: [[run-settlement]] · [[agent-event-stream]] · [[turn-loop]] · [[pi--run-settlement|pi]] · [[abort-propagation]] · [[codex--abort-propagation|codex]]
