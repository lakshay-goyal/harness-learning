---
type: failure
concepts: [step-budget-limit, steering-queue]
harnesses: [opencode]
---
**Symptom** — In the v2 runner a user message promoted mid-drain (steer or queued) inherited the provider-turn count already used by the previous request, so the agent hit "CRITICAL - MAXIMUM STEPS REACHED" and lost its tools right after receiving fresh instructions.

**Fix · [[opencode]]** `dc468bdcfd` 2026-06-23 "reset steps for promoted prompts" (#33452): promoting ≥1 admitted input resets `currentStep = 1` once per promotion batch (`packages/core/src/session/runner/llm.ts:186-196`); compaction transitions carry the step forward explicitly (`ContinueAfterCompaction { step }`, `llm.ts:152-166`). Terms in `CONTEXT.md` (Admitted Prompt, Prompt Promotion, Session Drain). Legacy still counts `step` per run, shared by queued messages (`packages/opencode/src/session/prompt.ts:1085,1132`).

**Lesson** — Step budgets belong to a user request, not to a drain or process run; reset them where new user input becomes model-visible.

Related: [[step-budget-limit]] · [[steering-queue]] · [[retry-counter-accumulates-across-turn]] (same class: per-turn counter spanning user turns) · [[opencode--step-budget-limit|opencode]]
