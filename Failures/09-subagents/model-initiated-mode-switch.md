---
type: failure
concepts: [plan-mode, agent-profiles]
harnesses: [opencode]
---
**Symptom** — Given a `plan_enter` tool ("If they explicitly mention wanting to create a plan ALWAYS call this tool first", `packages/opencode/src/tool/plan-enter.txt`), the model switched itself into plan mode in the middle of task execution and lost its write ability.

**Root cause** — A model-callable tool that reduces the model's own capabilities over-triggers on loosely related wording.

**Fix · [[opencode]]** — `fa559b0385` 2026-02-24 "temporarily disable plan enter tool to prevent unintended mode switches during task execution"; that commit dropped `PlanEnterTool` from the registry list, and `44f38193c0` 2026-04-10 (#21807, Effect refactor of `plan.ts`) deleted its definition (only `packages/opencode/src/tool/plan-enter.txt` remains); still disabled at HEAD `ecc4916b5a` (≈7.5 months), although the build agent keeps `plan_enter: allow` (`packages/opencode/src/agent/agent.ts:146-149`).

**Lesson** — Capability-reducing mode switches should be human-initiated; model-callable exits that ask for approval (`plan_exit`) are safer than model-callable entries.

Related: [[plan-mode]] · [[agent-profiles]] · [[ask-user-tool]] · [[opencode--plan-mode|opencode]]
