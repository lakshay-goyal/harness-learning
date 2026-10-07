---
type: concept
stage: tools
tier: must-have
aliases: [todowrite, todoread, Todo.Service, todo list, update_plan, PlanHandler, EventMsg::PlanUpdate, "Plan updated", todo tool, todo_write, planning tool, checklist tool, plan-checklist-tool]
harnesses: [opencode, codex]
---
A tool through which the model maintains a structured todo list that the harness persists and renders.

## Why
- Long multi-step tasks drift: without an explicit list the model forgets steps or declares completion early.
- The user sees progress in the UI instead of parsing prose.
- The list survives compaction as tool-call history, while prose plans may be summarized away.
- Long tasks drift without a visible checklist; users want progress visibility without the model re-printing its plan in prose.
- Re-printing the plan after each update wastes output tokens — codex prompt: "Do not repeat the full contents of the plan after an `update_plan` call — the harness already displays it." (`81b148bda2`, `codex-rs/protocol/src/prompts/base_instructions/default.md:58`).
- Models trained with the tool call it in harnesses that lack it ([[foreign-harness-tool-hallucination]]); models also over-use it on trivial tasks (codex softened "At the start of the task, call `update_plan`" → "any nontrivial task", `a6139aa003`).
- A checklist tool collides with a plan *mode* (proposal before execution) — codex hard-rejects it in Plan mode ([[plan-mode]], [[mode-state-confusion]]).

## Design space
- **Write-only full-list replace, echoed back as the result** (opencode, after `todoread` was removed) vs separate read tool vs item-level patch ops.
- States: pending / in_progress / completed / cancelled with exactly one in_progress (opencode).
- Cadence driven by prompt text ("3+ distinct steps", "When in doubt, use it") vs harness reminders.
- Per-model gating: hide for models that misuse it (opencode toggled for qwen and GPT, both reverted) → [[todo-tool-usage-calibration]].
- Scope: per session; denied to subagents by default (opencode).
- Absent; model keeps a plan in prose or a file (pi, [[no-todo-tool]]).
- **No todo tool; plan in prose or a file** (pi: [[no-todo-tool]]).
- **Stateless UI-event tool, state = last call's args in transcript** (codex `update_plan`).
- Stateful todo store the harness can re-inject after compaction (not in codex; unverified elsewhere).
- Behavioral guidance in tool description vs **in the system prompt** (codex moved it out, `30ee24521b`) → [[tool-description-design]], [[dynamic-tool-guidelines]].
- Default on and heavily prompted (codex 2025-07 → 2026-08) vs **opt-in, guidance stripped when off** (codex since `a9519cbcdd` 2026-08-31).
- Invariants enforced in schema/description (codex: at most one `in_progress`, status enum) vs in code.
- (folded from `plan-checklist-tool`, codex framing) A no-op "plan" tool whose only effect is publishing the model's step list (pending / in_progress / completed) to the UI as an event; the plan state lives only in the transcript (the call arguments), not in harness state.

## Implementations
- [[opencode--task-list-tool|opencode]] — `todowrite` persists the full list via `Todo.update`, title `N todos`; `general` agent and child sessions deny it unless allowed.
- [[codex--task-list-tool|codex]] — `update_plan(plan[{step,status}], explanation?)` → `EventMsg::PlanUpdate`, returns "Plan updated"; opt-in via `tools.update_plan.enabled`; refused in Plan mode.

## Failures
- [[todo-tool-usage-calibration]]
- [[prompt-names-unavailable-tools]]
- [[foreign-harness-tool-hallucination]] (pi side: Codex-trained models call `update_plan`)
- [[tool-description-lies-about-async]] (codex variant: behavioural prompting in the update_plan tool schema)
- (04) [[mode-state-confusion]]

## Tradeoffs
- [[todo-tool-vs-none]]

## Related
[[no-todo-tool]] · [[per-model-system-prompt]] · [[model-specific-toolset]] · [[tool-description-design]] · [[task-owned-subagent]]
[[plan-mode]] · [[ask-user-tool]] · [[dynamic-tool-guidelines]] · [[tool-description-design]] · [[no-todo-tool]] · [[guideline-softening]] · [[minimal-default-toolset]]
