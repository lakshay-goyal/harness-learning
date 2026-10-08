---
type: failure
concepts: [plan-mode, task-owned-subagent, permission-ruleset]
harnesses: [opencode]
---
**Symptom** — In plan mode (edits denied), the model delegated the change to the edit-capable `general` subagent through the `task` tool, and the files were edited anyway.

**Root cause** — Plan's edit denies live on the **agent** ruleset; a child session got only the parent **session's** denies plus its own agent's (permissive) rules, so the restriction did not follow the delegation.

**Fix · [[opencode]]** — three steps:
- `b8ca71d309` 2026-05-09 "subagent inherits parent agent's deny rules (Plan Mode security bypass)" (#26597); regression test `packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts`.
- `c4e676b8a0` 2026-05-12 "preserve subagent self permissions" (#27201): inheriting every agent deny broke subagents' own grants; narrowed.
- `3ad6923c61` 2026-06-10 "let subagents use their own permissions" (#31696): inheritance of agent denies removed ("Parent agent restrictions only govern that agent; the subagent's own permissions determine its capabilities", `packages/opencode/src/agent/subagent-permissions.ts:4-13`); instead the plan agent denies `task: {general: deny}` (`packages/opencode/src/agent/agent.ts:165-167`).
- Residual (observed): plan may still call `explore`, which has `bash: allow` (`packages/opencode/src/agent/agent.ts:196-209`); plan itself also allows bash ([[read-only-mode-bypassed-via-shell]]).

**Lesson** — A restricted mode must constrain what it can delegate to; the simplest enforcement is denying the spawn of capable subagents, not trying to merge rulesets across agents.

Related: [[plan-mode]] · [[task-owned-subagent]] · [[permission-ruleset]] · [[agent-profiles]] · [[subagent-config-not-inherited]] · [[opencode--plan-mode|opencode]]
