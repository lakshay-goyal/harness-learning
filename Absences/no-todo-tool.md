---
type: absence
harnesses: [pi]
---
# no-todo-tool

**What's missing**
- No TodoWrite / update_plan equivalent and no task-list state in the core loop or prompt.

**Evidence of decision**
- `9066f58ca` (2025-11-12), README "To-Dos": "**pi does not and will not support built-in to-dos.** In my experience, to-do lists generally confuse models more than they help. If you need task tracking, make it stateful by writing to a file: TODO.md … The agent can read and update this file as needed. Using checkboxes keeps track of what's done and what remains. Simple, visible, and under your control."
- `3424550d2` (2025-12-17): "**No built-in to-dos.** They confuse models. Use a TODO.md file, or build your own with custom tools."
- `25cc5c7bf^:packages/coding-agent/README.md:545`: "**No built-in to-dos.** They confuse models. Use a TODO.md file, or build your own with extensions."
- After `25cc5c7bf` (2026-09-22) this rationale is **no longer documented at HEAD**. `README.md:19` names only sub-agents and plan mode.
- Codex-trained models hallucinate their native `update_plan`. The Codex bridge prompt had to say "❌ UPDATE_PLAN DOES NOT EXIST — NEVER use: update_plan, updatePlan, read_plan, readPlan, todowrite, todoread …" (`1650041a6`, 2026-01-04; dropped `6484ae279` 2026-01-16).
- Anthropic OAuth "stealth mode" maps tool names to Claude Code casing, including `TodoWrite`, which pi does not ship (`packages/ai/src/api/anthropic-messages.ts:95-118`) → [[provider-identity-shim]].

**Opt-in replacement**
- `packages/coding-agent/examples/extensions/todo.ts`: a `todo` tool plus `/todos`. State is stored in tool-result `details`, "which allows proper branching" (`todo.ts:1-11`; pattern doc `examples/extensions/README.md:197-213`) → [[branch-scoped-extension-state]].
- The plan-mode example tracks steps with `[DONE:n]` tags in assistant text (`plan-mode/utils.ts:154`) → [[no-plan-mode]].
- TODO.md in the repo, read and written with the normal tools.

**History**
- Stated 2025-11-12 and never reversed. The rationale text was lost in the 2026-09-22 docs refresh.
- The Codex bridge (`1650041a6` → `6484ae279`) shows the cost: models trained with a todo tool still try to call it.

**Implication**
- Task state that must survive branching lives in the session tree (tool-result `details`), not in harness memory. This is a pi-wide extension idiom.
- File-based plans survive compaction verbatim. A todo tool's state would need its own re-injection after compaction (unverified as a stated reason).
- The claim "todos confuse models" is anecdotal ("In my experience"). There is no eval in `packages/evals` backing it (unverified beyond grep).

**codex** — *present, now opt-in*: `update_plan` since `8828f6f082` 2025-07-29 ("experimental plan tool"), prompted heavily from 2025-07-31 (`6ce0a5875b` "Initial planning tool"); handler `codex-rs/core/src/tools/handlers/plan_spec.rs:43` → [[plan-checklist-tool]]. 2026-08-31 `a9519cbcdd` "Make the update_plan tool opt-in (#41744)": default-off with all bundled guidance stripped when disabled — a move toward pi's stance. Exec JSONL still has a `todo_list` item kind (`codex-rs/exec/src/exec_events.rs:13-36`).

Related: [[branch-scoped-extension-state]] · [[plugin-tools]] · [[provider-identity-shim]] · [[no-plan-mode]] · [[minimal-default-toolset]] · [[Absences]] · [[plan-checklist-tool]]
