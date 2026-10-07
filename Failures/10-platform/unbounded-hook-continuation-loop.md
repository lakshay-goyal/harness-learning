---
type: failure
concepts: [extension-event-hooks, run-settlement]
harnesses: [pi]
---
**Symptom** — (documented hazard) An extension that unconditionally returns `continue:true` from `turn_end` / `agent_before_settle` makes the agent request another model turn forever.

**Root cause** — The settle boundary lets plugins request exactly ONE more request per boundary (`packages/coding-agent/src/core/extensions/types.ts:944-1003`), but nothing caps how many boundaries chain; pi has no max-turn cap anywhere (`packages/agent/src` has no numeric literal ≥10) → [[no-turn-cap]].

**Fix · [[pi]]** — No code guard; documented warning only (`packages/coding-agent/docs/extensions.md:117`). Continuation is rejected only when context can't continue (last role assistant, nothing queued) (`agent-session.ts:999-1036`).

**Lesson** — Any hook that can extend a run needs either a termination condition owned by the plugin or a harness-level budget; documenting the hazard leaves loops one bug away.

Related: [[extension-event-hooks]] · [[run-settlement]] · [[turn-lifecycle-hooks]] · [[pi--extension-event-hooks|pi]]
