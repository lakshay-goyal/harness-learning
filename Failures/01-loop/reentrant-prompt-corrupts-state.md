---
type: failure
concepts: [turn-loop, steering-queue]
harnesses: [pi]
---
**Symptom** — Calling `prompt()`/`continue()` while the agent was streaming raced and corrupted state; `Agent.reset()` during a run wiped the transcript mid-turn.

**Root cause** — No run-ownership guard on the Agent; mid-run input had no designated path.

**Fix · [[pi]]**
- `5ef3cc90d` 2026-01-02 — guard against concurrent `prompt()` (`packages/agent/CHANGELOG.md:739`); `d0a4c3702` same day splits queueing into `steer()`/`followUp()`.
- `1532c9994` 2026-08-06 — `reset()` rejects during active runs (#7717).
- HEAD: "Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion." (`packages/agent/src/agent.ts:372-380`), `continue()` guard (`:384-387`), `runWithLifecycle` guard (`:507-510`); session `prompt()` while streaming requires `streamingBehavior` (`packages/coding-agent/src/core/agent-session.ts:2012-2025`).

**Lesson** — Make the loop non-reentrant and force mid-run input through explicit queues.

Related: [[turn-loop]] · [[steering-queue]] · [[follow-up-queue]] · [[pi--turn-loop|pi]]
