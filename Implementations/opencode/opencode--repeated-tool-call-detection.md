---
type: implementation
harness: opencode
concept: repeated-tool-call-detection
commit: ecc4916b5a
files: [packages/opencode/src/session/processor.ts:29, packages/opencode/src/session/processor.ts:353-379, packages/opencode/src/agent/agent.ts:119-121, packages/opencode/src/session/processor.ts:200-202, packages/core/src/session/runner/llm.ts:55]
---
[[repeated-tool-call-detection]] in [[opencode]].

## Mechanism

### Legacy runtime — "doom loop" permission
- On every `tool-call` event the processor reads all parts of the **current assistant message** and takes the last `DOOM_LOOP_THRESHOLD` (`packages/opencode/src/session/processor.ts:353-356`).
- Trigger: exactly 3 parts, all `tool` parts with the same tool name, none `pending`, and `JSON.stringify(part.state.input) === JSON.stringify(input)` (`processor.ts:358-368`).
- Action: `permission.ask({permission:"doom_loop", patterns:[toolName], always:[toolName], metadata:{tool,input}})` against the agent ruleset (`processor.ts:370-377`). The human is the circuit breaker.
- Default ruleset `doom_loop: "ask"` (`packages/opencode/src/agent/agent.ts:121`); configurable to `allow`/`deny`. "Always" allows that tool name for the session.
- Reject → `PermissionV1.RejectedError` → tool fails → `ctx.blocked` → loop stops (`processor.ts:200-202`), unless `experimental.continue_loop_on_deny`.
- Scope: one assistant message = one model step, so it catches identical calls **within a step** (parallel batch); repeats across steps land in new assistant messages and are not compared (inferred from `MessageV2.parts(ctx.assistantMessage.id)`).

### v2 runtime
- Not implemented: "[ ] Bound provider retries and repeated identical tool calls" (`packages/core/src/session/runner/llm.ts:55`).

## Constants
| name | value | path:line |
|---|---|---|
| `DOOM_LOOP_THRESHOLD` | 3 | `packages/opencode/src/session/processor.ts:29` |
| default `doom_loop` action | `ask` | `packages/opencode/src/agent/agent.ts:121` |

## Evolution
- 2025-10-30 `d983b9485d` "add doom loop detection" (#3445).
- 2025-11-09 `4e549b1c05` user-configurable doom loop & external dir permissions; 2025-11-18 `47bfae52c0` permission checks fixed.

## Quirks / drift
- Exact `JSON.stringify` equality: key order or one changed argument defeats it; near-duplicates are missed.
- Within-step scope means the classic cross-step loop (same failing call every turn) is not caught by this guard (inference from code; no test seen).
- Prompt-side complement: per-model prompts tell the model not to repeat searches (04-prompting).

Failures: [[identical-tool-call-loop]].

Contrast: pi has no repetition guard and no turn cap ([[no-turn-cap]]); opencode routes the decision to the user through the permission channel.
