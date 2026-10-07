---
type: failure
concepts: [tool-call-gate, parallel-tool-execution, run-settlement]
harnesses: [pi]
---
**Symptom** — In multi-tool turns, `tool_call` handlers (policy/gate extensions) read `ctx.sessionManager` state that did not yet contain the assistant message whose tool calls they were judging.

**Root cause** — Tool interception lived in tool wrappers that ran while session event handling (persistence of `message_end`) was still in flight asynchronously; preflight raced the session's event queue.

**Fix · [[pi]]** — `63ac2df24` 2026-03-14 (closes #2113) "sync tool hooks with agent event processing": interception moved into agent-core `beforeToolCall`/`afterToolCall`, agent events drained before preflight, preflight sequential, execution parallel. Barrier later formalized by awaited listeners (`9022a5b5e` 2026-03-30; `packages/agent/src/agent.ts:565-612`): assistant `message_end` processing completes before tool preflight.

**Lesson** — A pre-execution gate must observe a state that already includes the decision it is gating; make persistence of the triggering message a barrier before policy runs.

Related: [[tool-call-gate]] · [[parallel-tool-execution]] · [[run-settlement]] · [[listeners-see-stale-agent-state]] · [[tool-result-persisted-before-tool-call]] · [[pi--tool-call-gate|pi]]
