---
type: absence
harnesses: [opencode]
---
# no-model-initiated-plan-entry

Plan mode exists, but the model cannot put itself into it.

**What's missing**
- No `plan_enter` tool. Entering the `plan` agent is a user action (agent switch). Leaving it is model-proposed via `plan_exit`, which asks the user Yes/No (`packages/opencode/src/tool/plan.ts:1-79`; flag `OPENCODE_EXPERIMENTAL_PLAN_MODE`, client `cli` only, `packages/opencode/src/tool/registry.ts:248`).
- `plan_enter` survives only as a permission key, default `deny`, still `allow` for `build` with no tool behind it (`packages/opencode/src/agent/agent.ts:127,149`).

**Evidence of decision**
- `0a3c72d678` (2026-01-13, #8281) "add plan mode with enter/exit tools".
- `fa559b0385` (2026-02-24) "core: temporarily disable plan enter tool to prevent unintended mode switches during task execution". The diff drops `PlanEnterTool` from the registry and comments out its import.
- `44f38193c0` (2026-04-10, #21807) deleted the remaining code. "Temporarily" became permanent.

**Implication**
- A tool that **reduces** the agent's own capability (edit → read-only) is a trap: the model calls it mid-task and silently loses the ability to finish. Capability-reducing mode switches should be human-initiated; capability-restoring ones (`plan_exit`) go through an approval.
- Failure → [[model-initiated-mode-switch]].

Related: [[plan-mode]] · [[agent-profiles]] · [[permission-ruleset]] · [[no-plan-mode]] · [[plan-mode-vs-none]] · [[opencode]] · [[Absences]]
