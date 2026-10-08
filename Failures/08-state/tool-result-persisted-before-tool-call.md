---
type: failure
concepts: [session-tree, run-settlement]
harnesses: [pi]
---
**Symptom** — With async extension event handlers installed, `toolResult` entries were written to the session JSONL **before** the assistant message containing their tool calls; on resume the persisted transcript violated tool_use → tool_result ordering (#1717).

**Root cause** — Session event handling (including `appendMessage` persistence) ran as fire-and-forget async handlers; persistence happened in handler-completion order rather than event order, so a slow `message_end` handler for the assistant lost the race to the tool results.

**Fix · [[pi]]**
- `dfc779faa` 2026-03-02 "serialize session event handling to preserve message order (fixes #1717)": promise-chain `_agentEventQueue` (`packages/coding-agent/src/core/agent-session.ts:223,317-407` at that commit).
- `9022a5b5e` 2026-03-30 — `Agent.subscribe()` listeners **awaited** and given the abort signal; assistant `message_end` processing is a barrier before tool preflight (`packages/agent/src/agent.ts:565-612`).
- `32bcdc973` 2026-05-19 — event queue + retry promise deleted, replaced by the synchronous post-run driver (`_runAgentPrompt`/`_handlePostAgentRun`).

**Lesson** — Persistence must be strictly ordered with the event stream; await listeners (or serialize them) rather than fire-and-forget, and make "message persisted" a barrier before anything that depends on it.

Related: [[session-tree]] · [[run-settlement]] · [[listeners-see-stale-agent-state]] · [[pre-tool-hook-sees-stale-state]] · [[pi--session-tree|pi]]
