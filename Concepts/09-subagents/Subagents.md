---
type: group
group: 09-subagents
---
Scope: delegating work to other agent contexts — subprocess subagents, task-owned child conversations under structured concurrency, and context handoff into fresh sessions.

## Concepts
- [[subagent-as-subprocess]] — Delegation by spawning the same CLI headless (json/print, no session) with a role prompt; output contract for the reader.
- [[task-owned-subagent]] — Subagent = child conversation owned by a tool task under structured concurrency; abort flows down.
- [[session-handoff]] — Transfer distilled context into a fresh session instead of compacting.

Absence: [[no-subagents-core]].
Failures: [[Subagents Failures]].
