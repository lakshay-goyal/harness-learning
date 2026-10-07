---
type: failure
concepts: [follow-up-queue, steering-queue, run-settlement]
harnesses: [pi]
---
**Symptom** — Queued steering/follow-up messages stayed stuck: after a threshold auto-compaction ended the run nothing restarted it (#1312); follow-ups queued by `agent_end` handlers (e.g. extension `sendUserMessage`) waited until the next user message (#5115).

**Root cause** — The queue check happened inside the low-level loop before the end-of-run work (compaction, `agent_end` handlers) that could enqueue or block; no "agent would stop" exit re-checked the queues afterwards.

**Fix · [[pi]]**
- `b050c582a` 2026-02-06 — resume queued messages after auto-compaction (originally `setTimeout(() => agent.continue(), 100)`); `Agent.continue()` with assistant tail drains one steering then one follow-up batch (`packages/agent/src/agent.ts:384-407`).
- `a29a7902e` 2026-05-28 (PR #5115, merge `8e77f8797`) — drain follow-ups queued during `agent_end`: `_handlePostAgentRun` ends with `hasQueuedMessages()` (`packages/coding-agent/src/core/agent-session.ts:1887-1889`).
- `32bcdc973` 2026-05-19 — timer replaced by awaited driver loop; before-settle boundary also continues when queued (`agent-session.ts:1893,1905`).

**Lesson** — Every "agent would stop" exit must re-check input queues *after* running end-of-run hooks.

Related: [[follow-up-queue]] · [[steering-queue]] · [[run-settlement]] · [[side-phase-input-lost]] · [[pi--follow-up-queue|pi]]
