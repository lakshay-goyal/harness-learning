---
type: failure
concepts: [turn-loop, agent-event-stream]
harnesses: [pi]
---
**Symptom** — Tools emitting progress callbacks after they resolved produced stale `tool_execution_update` events after `tool_execution_end`.

**Root cause** — The `onUpdate` callback outlived the `execute()` invocation; async tools commonly fire late callbacks.

**Fix · [[pi]]** — `daab056ac` 2026-06-12 — `acceptingUpdates` latch: updates after the tool settles are ignored; pending update promises awaited before returning (`packages/agent/src/agent-loop.ts:826-857`) (#5573).

**Lesson** — Scope callbacks to the invocation; per-tool event streams need a settled flag.

Related: [[turn-loop]] · [[agent-event-stream]] · [[parallel-tool-execution]] · [[pi--turn-loop|pi]]
