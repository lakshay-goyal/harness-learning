---
type: group
group: 09-subagents
---
Scope: delegating work to other agent contexts — subprocess subagents, task-owned child conversations under structured concurrency, in-process agent trees with mailboxes, internal delegate sessions (review), remote/voice delegation, context handoff into fresh sessions — and the agent profiles / modes that delegation and mode switches select.

## Concepts
- [[subagent-as-subprocess]] — Delegation by spawning the same CLI headless (json/print, no session) with a role prompt; output contract for the reader.
- [[task-owned-subagent]] — Subagent = child conversation owned by a tool task under structured concurrency; abort flows down.
- [[session-handoff]] — Transfer distilled context into a fresh session instead of compacting.
- [[plan-mode]] — Read-mostly agent profile; edits denied except to a plan file; explicit approval switches to an executing agent.
- [[agent-profiles]] — Named bundles of prompt, model, sampling, permission ruleset, step cap and mode (primary/subagent/all), switched by the user or delegated to by the model.

### Agent trees (in-process)
- [[in-process-subagent-threads]] — Sub-agents as threads in a shared thread manager, path-addressed, driven by spawn/message/wait/interrupt tools.
- [[subagent-result-mailbox]] — Child outcomes delivered asynchronously as typed queue-only mail; wait tool reports activity, not content.
- [[subagent-config-inheritance]] — Explicit rule set for what a child copies from the parent's live turn vs role/defaults.
- [[subagent-concurrency-limits]] — Thread/depth caps and LRU unloading of idle children.
- [[agent-message-board]] — Shared channel/thread board for all agents of a tree; notifications to active recipients only.

### Internal & external delegation
- [[review-subagent]] — One-shot locked-down child with a rubric as base instructions; JSON verdict folded back into the parent.
- [[delegate-session-runner]] — Generic in-process child-session primitive used by internal features (review, approvals, memory).
- [[cloud-task-delegation]] — Hand a task to a hosted environment (best-of-N), then apply a diff locally.
- [[voice-frontend-delegation]] — Realtime voice model fronts the user and delegates execution to the coding agent.

Absence: [[no-subagents-core]] · [[no-subagent-depth-limit-v2]].
Tradeoff: [[builtin-subagents-vs-none]].
Failures: [[Subagents Failures]].
