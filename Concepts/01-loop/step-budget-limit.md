---
type: concept
stage: loop
tier: variant
aliases: [turn-cap, step-limit-wrapup, max steps, maxSteps, MAX_STEPS_PROMPT, isLastStep, "CRITICAL - MAXIMUM STEPS REACHED", "agent.steps", StepLimitExceededError]
harnesses: [opencode]
---
A per-agent cap on model steps per run. On the last step the harness asks for a final answer, and forbids tools either by prompt or by `toolChoice:none`.

## Why
- Without a cap a looping model burns tokens until the user notices ([[identical-tool-call-loop]]).
- A hard error at the cap throws away the run; a forced wrap-up keeps a summary of work done and remaining tasks.
- An invented default cap kills legitimate long tasks ([[rewrite-introduces-unrequested-limits]]).
- A budget counted per drain instead of per user request starves new instructions ([[step-budget-not-reset-on-new-input]]).

## Design space
- **Default**: none, opt-in per agent (opencode) · hard default (opencode v2 briefly, 25) · none at all ([[no-turn-cap]], pi).
- **At the cap**: error the run (opencode v2 before `4f1a9d7aef`) · forced text-only wrap-up (opencode).
- **Enforcement**: prompt only, tools still offered (opencode legacy since `fed4776451`) · tools removed + `toolChoice: "none"` (opencode v2) · reject any tool call emitted anyway (opencode v2).
- **Wrap-up channel**: trailing assistant-role prefill (opencode) · system text (opencode `40eb8b93e1`, later removed) · user reminder.
- **Budget scope**: per run/drain (opencode legacy) · per admitted user request, reset on prompt promotion (opencode v2 `dc468bdcfd`).
- **Owner**: SDK step cap (`maxSteps: 1000`, opencode 2025-06) · harness loop.

## Implementations
- [[opencode--step-budget-limit|opencode]] — `agent.steps` (opt-in); last step appends assistant `MAX_STEPS_PROMPT`; v2 also strips tools and sets `toolChoice: "none"`.

## Failures
- [[step-limit-enforced-only-by-prompt]]
- [[rewrite-introduces-unrequested-limits]]
- [[step-budget-not-reset-on-new-input]]

## Tradeoffs
- [[turn-cap-vs-none]]

## Related
[[turn-loop]] · [[repeated-tool-call-detection]] · [[steering-queue]] · [[no-turn-cap]] · [[agent-profiles]] · [[ephemeral-reminder-injection]] · [[run-settlement]]
