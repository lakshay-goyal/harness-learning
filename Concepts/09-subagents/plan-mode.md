---
type: concept
stage: subagents
tier: candidate
aliases: [plan agent, plan_exit, plan_enter, plan.txt, plan-mode.txt, build-switch.txt, OPENCODE_EXPERIMENTAL_PLAN_MODE, ".opencode/plans"]
harnesses: [opencode]
---
A read-mostly agent profile in which edits are denied except to a plan file. An explicit approval switches to an executing agent.

## Why
- Users want the model to investigate and propose before touching files; "please don't edit yet" in prose is not reliably honored.
- A plan written to a file survives the switch and becomes the executing agent's brief ("The plan at X has been approved, you can now edit files. Execute the plan", `packages/opencode/src/tool/plan.ts:67`).
- Each enforcement gap became a bypass: shell writes ([[read-only-mode-bypassed-via-shell]]), delegation to an edit-capable subagent ([[read-only-mode-bypass-via-subagent]]), stale restrictions after leaving the mode ([[stale-mode-reminder-persists]]), the model switching itself in ([[model-initiated-mode-switch]]).

## Design space
- **Mechanism**: prompt-only reminder vs permission-scoped agent profile + reminder (opencode) vs tool-set removal.
- **What is denied**: edit-class tools except plan files (opencode) vs also shell (opencode: `bash` stays allowed; read-only shell is prompt-enforced only) vs a read-only shell parser.
- **Delegation**: subagents inherit the mode's restrictions (opencode 2026-05, reverted) vs deny specific edit-capable subagents from the plan agent (opencode HEAD, `task: {general: deny}`) vs no subagents in plan mode.
- **Entry**: user-only switch (opencode HEAD) vs model-callable enter tool (opencode `plan_enter`, disabled `fa559b0385`).
- **Exit**: user switches agent vs model-called exit tool that asks Yes/No and injects a synthetic build message (opencode `plan_exit`, experimental flag + CLI client).
- **Mode-change messaging**: inject an explicit "mode changed" reminder on the way out (opencode `build-switch.txt`).
- **Workflow script**: none vs phased workflow (explore → design → review → write plan → exit) copied from Claude Code (opencode experimental `plan-mode.txt`).
- **Absent**: pi ships no plan mode; example extension only ([[no-plan-mode]]).

## Implementations
- [[opencode--plan-mode|opencode]] — legacy `plan` agent: `edit: {"*": deny, ".opencode/plans/*.md": allow}`, `task: {general: deny}`, plan.txt reminder appended to the last user message each turn; experimental `plan_exit` tool; v2 core `plan` agent has the edit denies but no task tool exists in v2 yet.

## Failures
- [[read-only-mode-bypass-via-subagent]]
- [[read-only-mode-bypassed-via-shell]]
- [[stale-mode-reminder-persists]]
- [[model-initiated-mode-switch]]

## Tradeoffs
- [[plan-mode-vs-none]]

## Related
[[agent-profiles]] · [[permission-ruleset]] · [[task-owned-subagent]] · [[ephemeral-reminder-injection]] · [[ask-user-tool]] · [[no-plan-mode]] · [[opencode]]
