---
type: failure
concepts: [run-settlement, auto-retry-backoff]
harnesses: [pi]
---
**Symptom** — `session.prompt()` resolved before an auto-retry finished (incl. retries whose response made tool calls): print/RPC callers exited mid-retry or saw stale idle state; the next prompt threw "Agent is already processing"; tool results were lost.

**Root cause** — "Run done" was inferred from message events: the retry promise was created inside the async, serialized event processor (slow earlier events delayed `agent_end` handling so `waitForRetry()` missed it), and was resolved on the first successful `message_end` even when that response had tool calls; continuation was fire-and-forget.

**Fix · [[pi]]**
- `890329907` 2026-03-02 — create retry promise synchronously at `agent_end` dispatch (`packages/coding-agent/src/core/agent-session.ts:321,337` at that commit) (#1726).
- `8a0529ed9` 2026-03-20 — `_resolveRetry()` moved from `message_end` to `agent_end` (#2440); 0.65.0 "retried agent runs wait for the full retry cycle".
- `9022a5b5e` 2026-03-30 — awaited `Agent.subscribe()` listeners; `agent_end` no longer the idle boundary.
- `32bcdc973` + `c685b2736` 2026-05-19 — deleted `_agentEventQueue`/`_retryPromise`; synchronous driver `_runAgentPrompt`/`_handlePostAgentRun` (`agent-session.ts:1821-1890`); `agent_end.willRetry` computed synchronously (`:1197-1211`).
- `e9fa5a68a` 2026-07-09 — explicit `agent_settled` event (#6363).

**Lesson** — Model "run settled" as an explicit lifecycle state decided synchronously by one driver; never infer idle from message events or async handlers.

Related: [[run-settlement]] · [[auto-retry-backoff]] · [[pi--run-settlement|pi]]
