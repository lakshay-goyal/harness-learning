---
type: failure
concepts: [task-owned-subagent]
harnesses: [opencode]
---
**Symptom** — Subagents could spawn subagents without limit (whenever a subagent's own ruleset allowed `task`), fanning out recursively.

**Root cause** — The only guard was a default `task: deny` added to child sessions unless the subagent's ruleset mentioned `task`; nothing counted depth.

**Fix · [[opencode]]** — `285d315b4e` 2026-07-15 "limit subagent nesting depth" (#37124): walk the `parentID` chain; if depth ≥ `subagent_depth ?? 1` fail with `Subagent depth limit reached (N). Increase "subagent_depth" to allow nested subagents.` (`packages/opencode/src/tool/task.ts:104-117`). Default 1 = subagents cannot spawn subagents.

**Lesson** — Cap delegation depth by default and make deeper trees an explicit opt-in.

Related: [[task-owned-subagent]] · [[agent-profiles]] · [[opencode--task-owned-subagent|opencode]]
