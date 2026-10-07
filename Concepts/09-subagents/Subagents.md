---
type: group
group: 09-subagents
---
Scope: delegating work to other agent contexts — subprocess subagents, task-owned child conversations under structured concurrency, context handoff into fresh sessions — and the agent profiles / modes that delegation and mode switches select.

## Concepts
- [[subagent-as-subprocess]] — Delegation by spawning the same CLI headless (json/print, no session) with a role prompt; output contract for the reader.
- [[task-owned-subagent]] — Subagent = child conversation owned by a tool task under structured concurrency; abort flows down.
- [[session-handoff]] — Transfer distilled context into a fresh session instead of compacting.
- [[plan-mode]] — Read-mostly agent profile; edits denied except to a plan file; explicit approval switches to an executing agent.
- [[agent-profiles]] — Named bundles of prompt, model, sampling, permission ruleset, step cap and mode (primary/subagent/all), switched by the user or delegated to by the model.

Absence: [[no-subagents-core]].
Failures: [[Subagents Failures]].
