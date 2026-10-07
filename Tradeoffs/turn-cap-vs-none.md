---
type: tradeoff
concepts: [turn-loop, step-budget-limit, repeated-tool-call-detection, turn-lifecycle-hooks, abort-propagation]
harnesses: [pi, opencode]
---
# turn-cap-vs-none

**Axis**: what stops a runaway agent loop: a step budget and loop detector, or only the human, retries and the context window?

| option | pi | opencode | evidence |
|---|---|---|---|
| No cap anywhere; loop is `while (true)` | ✅ zero numeric literals ≥ 10 in `packages/agent/src` | legacy default (`agent.steps ?? Infinity`) | pi: `packages/agent/src/agent-loop.ts:179` → [[no-turn-cap]]; opencode `packages/opencode/src/session/prompt.ts:1178` |
| Per-agent step budget, graceful wrap-up | hook-buildable (`finishTurn` → `{action:"end"}`) | ✅ opt-in `steps`; last step appends `MAX_STEPS_PROMPT` ("Tools are disabled"). Legacy still sends tools (prose-only); v2 adds `toolChoice: "none"` | `packages/opencode/src/session/prompt.ts:1279-1285`; `packages/core/src/session/runner/llm.ts:202-222` → [[step-budget-limit]] |
| Repeated identical call guard | ❌ | ✅ legacy: 3 identical calls → `doom_loop` permission ask; ❌ in v2 | `packages/opencode/src/session/processor.ts:29,356-373`; `packages/core/src/session/runner/llm.ts:55` → [[repeated-tool-call-detection]] |
| History | never had one | SDK `maxSteps: 1000` (`fa1266263d` 2025-06-14) removed `e2fac991dc` 2025-08-09; per-agent cap `40eb8b93e1` 2025-12-05 | [[Constants]] 01-loop |
| Bounded recovery loops instead | retries 3 (60 s cap), overflow recovery once per turn | retries 5 (uncapped until `c78986831c` 2026-08-11); v2 overflow recovery once | pi `packages/coding-agent/src/core/settings-manager.ts:1026`; opencode `packages/opencode/src/session/retry.ts:31` |

**Shared insight**: both bound the *recovery* loops (retry, overflow) and leave the *work* loop unbounded by default. Neither ships a default step cap.

**When each wins**
- **No cap (pi)**: interactive use with Esc; long legitimate tasks are never cut off mid-edit; embedders add budgets via hooks.
- **Opt-in budget + loop guard (opencode)**: subagents and unattended runs, where a forced summary beats a silent loop; the 3-identical-call ask catches the cheapest failure mode. Pair the wrap-up prompt with real tool removal, or the model keeps calling tools that the prompt says are disabled.

Related: [[no-turn-cap]] · [[auto-retry-backoff]] · [[identical-tool-call-loop]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
