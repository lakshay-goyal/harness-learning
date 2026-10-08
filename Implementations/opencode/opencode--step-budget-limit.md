---
type: implementation
harness: opencode
concept: step-budget-limit
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:1178-1179, packages/opencode/src/session/prompt.ts:1279-1285, packages/core/src/session/runner/max-steps.ts:1-16, packages/core/src/session/runner/llm.ts:186-222, packages/core/src/session/runner/llm.ts:252-255, packages/core/src/v1/config/agent.ts:34-37]
---
[[step-budget-limit]] in [[opencode]].

## Mechanism
- Per-agent config `steps` (legacy alias `maxSteps`, `packages/core/src/v1/config/agent.ts:34-37`). **Opt-in**: default is no cap.
- Wrap-up text `MAX_STEPS_PROMPT`: "CRITICAL - MAXIMUM STEPS REACHED … Tools are disabled until next user input. Respond with text only." plus required sections (limit reached, work done, remaining tasks, recommendations) (`packages/core/src/session/runner/max-steps.ts:1-16`). Injected as a trailing **assistant-role** message (prefill position), not user or system.

### Legacy runtime — prompt-only enforcement
- `maxSteps = agent.steps ?? Infinity`; `isLastStep = step >= maxSteps` (`packages/opencode/src/session/prompt.ts:1178-1179`).
- Last step appends `{role:"assistant", content: MAX_STEPS_PROMPT}` but still passes the full `tools` and `toolChoice` only for structured output (`prompt.ts:1279-1285`). If the model calls a tool anyway the loop continues and every later step is again "last".
- `step` counts per run, so queued/steered user messages share the budget.

### v2 runtime — structural enforcement
- `isLastStep = agent.info?.steps !== undefined && currentStep >= agent.info.steps`; on the last step tools are **not materialized**, `tools: []`, `toolChoice: "none"`, plus the same assistant-role prompt (`packages/core/src/session/runner/llm.ts:202-222`).
- Any tool call still emitted is failed "Tools are disabled after the maximum agent steps" and does not set `needsContinuation` (`llm.ts:252-255`).
- Budget belongs to a user request: promoting ≥1 steer/queued input resets `currentStep = 1` (`llm.ts:186-196`); compaction transitions carry the step forward (`ContinueAfterCompaction { step }`).

## Constants
| name | value | path:line |
|---|---|---|
| default cap | none (`Infinity` legacy; `undefined` v2) | `packages/opencode/src/session/prompt.ts:1178`; `packages/core/src/session/runner/llm.ts:202` |
| `MAX_STEPS_PROMPT` role | assistant | `packages/opencode/src/session/prompt.ts:1281`; `packages/core/src/session/runner/llm.ts:220` |

## Evolution
- 2025-06-14 `fa1266263d` AI SDK `maxSteps: 1000` (SDK-owned cap) → 2025-08-09 `e2fac991dc` `stopWhen: stepCountIs(1000)` replaced by a custom predicate.
- 2025-12-05 `40eb8b93e1` per-agent `maxSteps` + `max-steps.txt`; set `toolChoice: "none"` and pushed the text into system.
- 2025-12-14 `fed4776451` "LLM cleanup" removed `toolChoice: "none"` and the system push → legacy enforcement became prompt-only.
- 2026-06-20 `4f1a9d7aef` v2 had shipped `const MAX_STEPS = 25` with an in-code "QUESTION: Did this exist previously, or did we add this limit?" and failed runs with `StepLimitExceededError`; fix = no default cap, wrap-up instead of error; text moved into core.
- 2026-06-23 `dc468bdcfd` v2 resets steps for promoted prompts.

## Quirks / drift
- Failure: [[step-limit-enforced-only-by-prompt]].
- Legacy prompt claims "Tools are disabled" while tools are still offered (no reported incident; inferred from code).
- v2 sends `toolChoice: "none"` → Anthropic lowering drops all tools (`packages/llm/src/protocols/anthropic-messages.ts:515-523`) → tool-prefix cache likely invalidated on that turn (inference, unverified).
- Assistant-role prefill is rejected by some providers/models that disallow trailing assistant messages (unverified for opencode).

Failures: [[rewrite-introduces-unrequested-limits]] · [[step-budget-not-reset-on-new-input]].

Contrast: pi has no cap at all ([[no-turn-cap]]); opencode makes it opt-in per agent and turns the cap into a forced text wrap-up instead of an error.
