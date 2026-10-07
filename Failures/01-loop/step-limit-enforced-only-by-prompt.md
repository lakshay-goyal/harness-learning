---
type: failure
concepts: [step-budget-limit, ephemeral-reminder-injection]
harnesses: [opencode]
---
**Symptom** On the last allowed step, the legacy runtime tells the model "Tools are disabled until next user input. Respond with text only." (`packages/core/src/session/runner/max-steps.ts:3`). That text is appended as an assistant message (`packages/opencode/src/session/prompt.ts:1281`). The same request still carries the full `tools` set and leaves `toolChoice` unset (`packages/opencode/src/session/prompt.ts:1278-1286`), so the model can keep calling tools past the cap.

**History · [[opencode]]**
- `40eb8b93e1` 2025-12-05 "add max steps for supervisor and sub-agents" (#4062) enforced the cap in code with `toolChoice: isLastStep ? "none" : undefined`.
- `fed4776451` 2025-12-14 "LLM cleanup" (#5462) moved request building into `session/llm.ts` and dropped the `toolChoice` line. The prompt text stayed. Nothing has re-added it in the legacy runtime since.
- **Fix (v2 only)**: `packages/core/src/session/runner/llm.ts:222` sets `toolChoice: isLastStep ? "none" : undefined`. Configured agent step limits have been honored since `4f1a9d7aef` 2026-06-20.

**Lesson** Enforce a limit in the request (tool removal or `toolChoice`), not only in prompt text. Add a regression test, because refactors drop request fields while the prompt still claims enforcement.

Related: [[step-budget-limit]] · [[opencode--step-budget-limit|opencode]] · [[turn-cap-vs-none]] · [[tool-description-drifts-from-implementation]]
