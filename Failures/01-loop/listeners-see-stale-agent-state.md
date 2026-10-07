---
type: failure
concepts: [run-settlement, agent-event-stream]
harnesses: [pi]
---
**Symptom** — Event handlers on `message_end` (and other events) read agent state that did not yet include the event's effect.

**Root cause** — Events were emitted before the agent state reducer applied them.

**Fix · [[pi]]** — `2b0aa5ed8` 2025-12-09 "update state before emitting events": reducer runs first, then listeners (`packages/agent/src/agent.ts:565-603`). Later `9022a5b5e` 2026-03-30 made listeners awaited so assistant `message_end` processing is a barrier before tool preflight (`agent.ts:565-612`).

**Lesson** — Reduce state, then notify; make listener completion part of the lifecycle when later phases depend on it.

Related: [[run-settlement]] · [[agent-event-stream]] · [[turn-loop]] · [[pi--run-settlement|pi]]
