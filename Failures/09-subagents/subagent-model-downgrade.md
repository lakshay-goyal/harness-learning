---
type: failure
concepts: [task-owned-subagent]
harnesses: [codex]
---
**Symptom** — The `spawn_agent` description said "A mini model can solve many tasks faster than the main model", so gpt-5.4 "would occasionally pick 5.2 or lower as sub-agents" (`54bd07d28c` body).

**Fix · [[codex]]**
- `54bd07d28c` 2026-04-20 "[codex] prefer inherited spawn agent model (#18701)" — phrase removed (test asserts absence); "Do not set the `model` field unless the user explicitly asks for a different model"; list framed "Available model overrides (optional; inherited parent model is preferred)", ≤ 5 models (`codex-rs/core/src/agent/child_config.rs:20`). Phrase had been added `932ff28183` 2026-03-04.

**Lesson** — Any model menu in a tool description is a suggestion; default to inheritance and say so.

Related: [[task-owned-subagent]] · [[subagent-config-not-inherited]] · [[subagent-config-inheritance]] · [[tool-description-design]] · [[codex--task-owned-subagent|codex]]
