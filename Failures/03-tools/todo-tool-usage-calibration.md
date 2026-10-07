---
type: failure
concepts: [task-list-tool, per-model-system-prompt, model-specific-toolset]
harnesses: [opencode]
---
**Symptom** — Todo-tool usage misfired by model family: non-Claude models misused the todo instructions, qwen and GPT models had the todo tools toggled off and back on, and Claude under-used todos when the "Always use TodoWrite" line was dropped.

**Root cause** — Todo cadence is model-specific; a single always-on rule over- or under-triggers depending on the family.

**Fix · [[opencode]]**
- `df03e182d2` 2025-07-29 "strip todo tool instructions from non anthropic models" (fallback prompt).
- `5cc44c872e` 2025-08-12 todo tools disabled for qwen → `1cea8b9e77` 2025-09-08 re-enabled.
- `795b845782` 2025-10-25 Claude Code sync dropped "IMPORTANT: Always use the TodoWrite tool…" → `22821744ef` 2025-10-28 re-added (`packages/opencode/src/session/prompt/anthropic.txt:96`).
- `3515b4ff7d` 2026-01-19 omitted for openai models → `d9f0287d74` 2026-01-21 added back.
- Residual: `general` subagent (no own prompt) gets anthropic.txt's "Always use the TodoWrite tool" while its permission denies `todowrite` → [[prompt-names-unavailable-tools]].

**Lesson** — Calibrate todo cadence per model family and A/B removals of "always" lines before shipping.

Related: [[task-list-tool]] · [[per-model-system-prompt]] · [[model-specific-toolset]] · [[no-todo-tool]] · [[opencode--task-list-tool|opencode]]
