---
type: concept
stage: tool-design
tier: candidate
aliases: [update_plan, PlanHandler, EventMsg::PlanUpdate, "Plan updated", todo tool, todo_write, planning tool, checklist tool]
harnesses: [codex]
---
A no-op "plan" tool whose only effect is publishing the model's step list (pending / in_progress / completed) to the UI as an event; the plan state lives only in the transcript (the call arguments), not in harness state.

## Why
- Long tasks drift without a visible checklist; users want progress visibility without the model re-printing its plan in prose.
- Re-printing the plan after each update wastes output tokens — codex prompt: "Do not repeat the full contents of the plan after an `update_plan` call — the harness already displays it." (`81b148bda2`, `codex-rs/protocol/src/prompts/base_instructions/default.md:58`).
- Models trained with the tool call it in harnesses that lack it ([[foreign-harness-tool-hallucination]]); models also over-use it on trivial tasks (codex softened "At the start of the task, call `update_plan`" → "any nontrivial task", `a6139aa003`).
- A checklist tool collides with a plan *mode* (proposal before execution) — codex hard-rejects it in Plan mode ([[plan-mode]], [[mode-state-confusion]]).

## Design space
- **No todo tool; plan in prose or a file** (pi: [[no-todo-tool]]).
- **Stateless UI-event tool, state = last call's args in transcript** (codex `update_plan`).
- Stateful todo store the harness can re-inject after compaction (not in codex; unverified elsewhere).
- Behavioral guidance in tool description vs **in the system prompt** (codex moved it out, `30ee24521b`) → [[tool-description-design]], [[dynamic-tool-guidelines]].
- Default on and heavily prompted (codex 2025-07 → 2026-08) vs **opt-in, guidance stripped when off** (codex since `a9519cbcdd` 2026-08-31).
- Invariants enforced in schema/description (codex: at most one `in_progress`, status enum) vs in code.

## Implementations
- [[codex--plan-checklist-tool|codex]] — `update_plan(plan[{step,status}], explanation?)` → `EventMsg::PlanUpdate`, returns "Plan updated"; opt-in via `tools.update_plan.enabled`; refused in Plan mode.

## Failures
- [[foreign-harness-tool-hallucination]] (pi side: Codex-trained models call `update_plan`)
- [[tool-description-lies-about-async]] (codex variant: behavioural prompting in the update_plan tool schema)
- (04) [[mode-state-confusion]]

## Related
[[plan-mode]] · [[structured-user-question-tool]] · [[dynamic-tool-guidelines]] · [[tool-description-design]] · [[no-todo-tool]] · [[guideline-softening]] · [[minimal-default-toolset]]
