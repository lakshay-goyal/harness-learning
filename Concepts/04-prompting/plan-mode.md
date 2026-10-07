---
type: concept
stage: messages
tier: candidate
aliases: [collaboration mode, collaboration-modes, ModeKind::Plan, ModeKind::Default, "<collaboration_mode>", "<proposed_plan>", PlanModeStreamState, collaboration-mode-templates, "/plan"]
harnesses: [codex]
---
A user-switchable collaboration mode in which the model explores read-only, interviews the user, and ends with one tagged, decision-complete plan block; the mode is a developer message (data), not a different tool set or sandbox.

## Why
- Users want "think and ask before touching anything" without hand-writing that instruction every time.
- Without an explicit mode the model treats imperative user phrasing ("just do it") as permission to edit, or asks questions before exploring ([[mode-state-confusion]]).
- A tagged final plan (`<proposed_plan>`) lets the client render it and offer "implement this plan".
- Rigid turn-shape rules over-constrain the model ([[imperative-guideline-over-compliance]]).

## Design space
- No plan mode; user asks in prose ✔ pi ([[no-plan-mode]]).
- **Prompt-only mode: developer fragment swapped per mode, appended on switch, with explicit "previous mode no longer active" text** ✔ codex.
- Hard enforcement via tools/sandbox (read-only sandbox, tool removal) vs **one hard check only** (`update_plan` rejected in Plan) ✔ codex.
- Mode-gated tools: blocking `request_user_input` only in Plan ✔ codex; history: rejected outside Plan/Pair (2026-01) → re-enabled in Default (2026-02-25).
- Mode set: Default / Plan ✔ codex; removed Pair Programming / Execute / Custom ([[no-collaboration-styles]]).
- Per-model mode text from the catalog (`model_messages.collaboration_modes`) ✔ codex → [[per-model-system-prompt]].
- Preset knobs per mode (Plan → reasoning effort Medium) ✔ codex.
- Plan output: one `<proposed_plan>` block per turn, revisions are complete replacements ✔ codex (`c3cb38eafb`).

## Implementations
- [[codex--plan-mode|codex]] — `codex-rs/collaboration-mode-templates/templates/plan.md` (9,184 B) / `default.md` (1,288 B) as `<collaboration_mode>` developer fragments in world state; `update_plan` hard error in Plan; `<proposed_plan>` parsed from stream; 26 commits to plan.md, 16 in its first two weeks.

## Failures
- [[mode-state-confusion]]
- [[imperative-guideline-over-compliance]]

## Related
[[message-role-layering]] · [[world-state-diff-injection]] · [[plan-checklist-tool]] · [[structured-user-question-tool]] · [[guideline-softening]] · [[dynamic-tool-guidelines]] · [[no-plan-mode]]
