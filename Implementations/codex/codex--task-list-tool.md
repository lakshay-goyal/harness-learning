---
type: implementation
harness: codex
concept: task-list-tool
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/plan_spec.rs:10-53, codex-rs/core/src/tools/handlers/plan.rs:22, codex-rs/core/src/tools/handlers/plan.rs:86-98, codex-rs/core/src/tools/spec_plan.rs:1228, codex-rs/core/src/config/mod.rs:2730-2736, codex-rs/prompts/src/update_plan_instructions.rs:4-60]
---
[[task-list-tool]] in [[codex]].

## Mechanism
- Schema (`codex-rs/core/src/tools/handlers/plan_spec.rs:10-53`): `plan: [{step: string "Task step text.", status: enum pending|in_progress|completed "Step status."}]` ("The list of steps", required), `explanation?: string` "Optional explanation for this plan update."
- Description: "Updates the task plan. Provide an optional explanation and a list of plan items, each with a step and status. At most one step can be in_progress at a time." (`plan_spec.rs:44-47`) — invariant stated, not enforced in code (no check found in `plan.rs`).
- Handler: rejects in Plan mode with "update_plan is a TODO/checklist tool and is not allowed in Plan mode"; else `session.send_event(EventMsg::PlanUpdate(args))` and returns "Plan updated" (`codex-rs/core/src/tools/handlers/plan.rs:22,86-98`). No harness-side plan store — the plan survives only as the call in history.
- Not parallel-safe (default) → exclusive lock ([[parallel-tool-execution]]).
- Gate: `config.update_plan_enabled` (`codex-rs/core/src/tools/spec_plan.rs:1228`) = `tools.update_plan.enabled` in config.toml, **false when absent** (`codex-rs/core/src/config/mod.rs:2730-2736`).
- When disabled, `without_update_plan_instructions` strips Codex-owned prompt sections by literal heading (`## Planning`, ``## `update_plan` ``, `## Plan tool`, `## Plan Mode vs update_plan tool`), lines starting "You have access to an `update_plan` tool" / "When `update_plan` is available…", "Progress visibility:" blocks, and bullets "- Use the plan tool …" / "- If you create a checklist or task list," (`codex-rs/prompts/src/update_plan_instructions.rs:4-60`); custom instructions untouched → [[dynamic-tool-guidelines]].

## Constants
| name | value | path:line |
|---|---|---|
| default enabled | `false` (opt-in) | `codex-rs/core/src/config/mod.rs:2730-2736` |
| status values | pending / in_progress / completed | `codex-rs/core/src/tools/handlers/plan_spec.rs:16` |
| max `in_progress` (prompted) | 1 | `codex-rs/core/src/tools/handlers/plan_spec.rs:46` |
| plan-tool skip threshold (gpt-5-codex prompt) | "roughly the easiest 25%" of tasks | `codex-rs/core/gpt_5_codex_prompt.md:24` |

## Evolution
- 2025-07-29 `8828f6f082` "Add an experimental plan tool (#1726)"; 2025-07-31 `6ce0a5875b` "Initial planning tool (#1753)" (prompted).
- 2025-08-04 `a6139aa003` "At the start of the task, call `update_plan`" → "At the start of any nontrivial task" (plan spam on trivial tasks).
- 2025-08-07 `81b148bda2` "Do not repeat the full contents of the plan after an `update_plan` call — the harness already displays it." (`default.md:58`).
- 2025-08-13 `30ee24521b` "remove behavioral prompting from update_plan tool def (#2261)" — "Skip a plan when…", "Planning steps are … like tasks or TODOs" moved into the main prompt ([[tool-description-lies-about-async]] codex variant).
- 2025-09-14 `916fdc2a37` gpt-5-codex: "Skip using the planning tool for straightforward tasks (roughly the easiest 25%)."
- 2026-01-29 `b0f1b9f622` side-branch rename `update_plan`→`todo_write` (keep alias) — never merged.
- 2026-05-29 `1c55bb2702` status converted to an enum ("Improve built-in tool schema docs").
- 2026-08-31 `a9519cbcdd` "Make the update_plan tool opt-in (#41744)": default off; bundled update_plan guidance removed from model, collaboration-mode, multi-agent, compaction, prewarm and goal-continuation prompts.
- 2026-01-28 `b0f1b9f622` / `8b8840047c` side-branch rename `update_plan` → `todo_write`; never merged to mainline (rejected experiment).

## Quirks
- Plan-mode is prompt-level except this one hard check ([[plan-mode]]).
- After `a9519cbcdd` the trained-in habit (model calling `update_plan`) meets "unsupported call: update_plan" when disabled ([[tool-error-as-result]]) — no recorded failure yet (unverified).

## Versus pi
pi has no todo tool by stance ([[no-todo-tool]]) and had to tell Codex-trained models "UPDATE_PLAN DOES NOT EXIST" ([[foreign-harness-tool-hallucination]]). codex itself moved toward pi's position by making the tool opt-in.
