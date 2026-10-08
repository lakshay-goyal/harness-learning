---
type: concept
stage: subagents
tier: must-have
aliases: [plan agent, plan_exit, plan_enter, plan.txt, plan-mode.txt, build-switch.txt, OPENCODE_EXPERIMENTAL_PLAN_MODE, ".opencode/plans", collaboration mode, collaboration-modes, ModeKind::Plan, ModeKind::Default, "<collaboration_mode>", "<proposed_plan>", PlanModeStreamState, collaboration-mode-templates, "/plan"]
harnesses: [opencode, codex]
---
A read-mostly agent profile in which edits are denied except to a plan file. An explicit approval switches to an executing agent.

## Why
- Users want the model to investigate and propose before touching files; "please don't edit yet" in prose is not reliably honored.
- A plan written to a file survives the switch and becomes the executing agent's brief ("The plan at X has been approved, you can now edit files. Execute the plan", `packages/opencode/src/tool/plan.ts:67`).
- Each enforcement gap became a bypass: shell writes ([[read-only-mode-bypassed-via-shell]]), delegation to an edit-capable subagent ([[read-only-mode-bypass-via-subagent]]), stale restrictions after leaving the mode ([[stale-mode-reminder-persists]]), the model switching itself in ([[model-initiated-mode-switch]]).
- Users want "think and ask before touching anything" without hand-writing that instruction every time.
- Without an explicit mode the model treats imperative user phrasing ("just do it") as permission to edit, or asks questions before exploring ([[mode-state-confusion]]).
- A tagged final plan (`<proposed_plan>`) lets the client render it and offer "implement this plan".
- Rigid turn-shape rules over-constrain the model ([[imperative-guideline-over-compliance]]).

## Design space
- **Mechanism**: prompt-only reminder vs permission-scoped agent profile + reminder (opencode) vs tool-set removal.
- **What is denied**: edit-class tools except plan files (opencode) vs also shell (opencode: `bash` stays allowed; read-only shell is prompt-enforced only) vs a read-only shell parser.
- **Delegation**: subagents inherit the mode's restrictions (opencode 2026-05, reverted) vs deny specific edit-capable subagents from the plan agent (opencode HEAD, `task: {general: deny}`) vs no subagents in plan mode.
- **Entry**: user-only switch (opencode HEAD) vs model-callable enter tool (opencode `plan_enter`, disabled `fa559b0385`).
- **Exit**: user switches agent vs model-called exit tool that asks Yes/No and injects a synthetic build message (opencode `plan_exit`, experimental flag + CLI client).
- **Mode-change messaging**: inject an explicit "mode changed" reminder on the way out (opencode `build-switch.txt`).
- **Workflow script**: none vs phased workflow (explore → design → review → write plan → exit) copied from Claude Code (opencode experimental `plan-mode.txt`).
- **Absent**: pi ships no plan mode; example extension only ([[no-plan-mode]]).
- No plan mode; user asks in prose ✔ pi ([[no-plan-mode]]).
- **Prompt-only mode: developer fragment swapped per mode, appended on switch, with explicit "previous mode no longer active" text** ✔ codex.
- Hard enforcement via tools/sandbox (read-only sandbox, tool removal) vs **one hard check only** (`update_plan` rejected in Plan) ✔ codex.
- Mode-gated tools: blocking `request_user_input` only in Plan ✔ codex; history: rejected outside Plan/Pair (2026-01) → re-enabled in Default (2026-02-25).
- Mode set: Default / Plan ✔ codex; removed Pair Programming / Execute / Custom ([[no-collaboration-styles]]).
- Per-model mode text from the catalog (`model_messages.collaboration_modes`) ✔ codex → [[per-model-system-prompt]].
- Preset knobs per mode (Plan → reasoning effort Medium) ✔ codex.
- Plan output: one `<proposed_plan>` block per turn, revisions are complete replacements ✔ codex (`c3cb38eafb`).

## Implementations
- [[opencode--plan-mode|opencode]] — legacy `plan` agent: `edit: {"*": deny, ".opencode/plans/*.md": allow}`, `task: {general: deny}`, plan.txt reminder appended to the last user message each turn; experimental `plan_exit` tool; v2 core `plan` agent has the edit denies but no task tool exists in v2 yet.
- [[codex--plan-mode|codex]] — `codex-rs/collaboration-mode-templates/templates/plan.md` (9,184 B) / `default.md` (1,288 B) as `<collaboration_mode>` developer fragments in world state; `update_plan` hard error in Plan; `<proposed_plan>` parsed from stream; 26 commits to plan.md, 16 in its first two weeks.

## Failures
- [[read-only-mode-bypass-via-subagent]]
- [[read-only-mode-bypassed-via-shell]]
- [[stale-mode-reminder-persists]]
- [[model-initiated-mode-switch]]
- [[mode-state-confusion]]
- [[imperative-guideline-over-compliance]]

## Tradeoffs
- [[plan-mode-vs-none]]

## Related
[[agent-profiles]] · [[permission-ruleset]] · [[task-owned-subagent]] · [[ephemeral-reminder-injection]] · [[ask-user-tool]] · [[no-plan-mode]] · [[opencode]]
[[message-role-layering]] · [[world-state-diff-injection]] · [[task-list-tool]] · [[ask-user-tool]] · [[guideline-softening]] · [[dynamic-tool-guidelines]] · [[no-plan-mode]]
