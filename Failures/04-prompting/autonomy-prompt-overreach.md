---
type: failure
concepts: [per-model-system-prompt, minimal-system-prompt]
harnesses: [opencode]
---
**Symptom** — GPT prompts modelled on Codex pushed persistence ("Persist until the task is fully handled end-to-end within the current turn … do not stop at analysis or partial fixes", `packages/opencode/src/session/prompt/gpt.txt:15-19`), which invites acting beyond what was asked (e.g. editing when asked to explain). The concrete incident is not in the commit (symptom unverified).

**Root cause** — An autonomy section tuned for a fully delegated agent applied to every request type.

**Fix · [[opencode]]**
- origin/v2 GPT extension first enumerated request types (answer/explain/review do not authorize edits; diagnose ≠ fix), then `a7b8174917` 2026-09-02 "replace GPT autonomy section with scope guidance (#46864)" cut it to one rule: "Do not infer authorization for work beyond the user's request. Assumptions that help you make progress are fine as long as they stay within the user's intent and the scope of the task." (`origin/v2:packages/core/src/plugin/system-prompt/gpt.txt:37-39`).
- Dev `gpt.txt` still carries the persistence section at HEAD.

**Lesson** — Pair every autonomy push with an explicit scope bound.

Related: [[per-model-system-prompt]] · [[minimal-system-prompt]] · [[guideline-softening]] · [[opencode--per-model-system-prompt|opencode]]
