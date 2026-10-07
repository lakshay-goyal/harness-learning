---
type: failure
concepts: [per-model-system-prompt]
harnesses: [codex]
---
**Symptom** — The model stopped at analysis/proposal, output a proposed solution as a message instead of implementing, gave up when tool calls failed, or stopped to ask permission it already had.

**Root cause** — Each new model generation defaulted to a more conversational, confirm-first style than the harness wanted.

**Fix · [[codex]]**
- `8dcbd29edd` 2025-11-13 GPT-5.1 "## Autonomy and Persistence": "Persist until the task is fully handled end-to-end within the current turn whenever feasible: do not stop at analysis or partial fixes", "it's bad to output your proposed solution in a message, you should go ahead and actually implement the change", "persevere even when function calls fail" (`codex-rs/core/gpt_5_1_prompt.md:29-33`, `:138`).
- `ed391d4dd2` 2026-09-03 GPT-6-Astra catalog text: "Do not stop at acknowledging capability (e.g. "Yes…"), proposing a plan, or offering to continue.", "The user gets very frustrated when you stop and ask for confirmation or permission, so make sure to explicitly explain why you need the confirmation", "Do not request permission again when the user has already authorized an action in an earlier turn." (`codex-rs/models-manager/models.json`, gpt-6-astra `instructions_template` lines 3-25).
- Counter-trend in the same text: "Do not introduce unsolicited warnings, disclaimers, approval flows, or safety/compliance checklists due to hypothetical risk."

**Lesson** — Each new model generation needed a stronger "bias to action" section; put permission-asking rules at the top of the newest prompt and re-evaluate them per model.

Related: [[per-model-system-prompt]] · [[over-validation-in-interactive-mode]] · [[approval-asked-in-prose]] · [[codex--per-model-system-prompt|codex]]
