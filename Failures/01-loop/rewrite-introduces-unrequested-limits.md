---
type: failure
concepts: [step-budget-limit, turn-loop]
harnesses: [opencode]
---
**Symptom** — After the v2 runtime port, long tasks died at an arbitrary 25 steps with `StepLimitExceededError`, a limit the old runtime never had. The port carried an in-code note "QUESTION: Did this exist previously, or did we add this limit?".

**Fix · [[opencode]]** `4f1a9d7aef` 2026-06-20 "honor configured agent step limits" (#33142): no default cap; only an agent's configured `steps` bounds the loop, and the last step becomes a text-only wrap-up (`tools: []`, `toolChoice: "none"`, assistant-role `MAX_STEPS_PROMPT`) instead of an error (`packages/core/src/session/runner/llm.ts:202-222`; `packages/core/src/session/runner/max-steps.ts:1-16`).

**Lesson** — A runtime rewrite must diff its behaviour constants against the old runtime; an unasked-for limit is a regression even when it looks like safety.

Related: [[step-budget-limit]] · [[turn-loop]] · [[rewrite-drops-test-coverage]] · [[opencode--step-budget-limit|opencode]]
