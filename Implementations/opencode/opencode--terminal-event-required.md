---
type: implementation
harness: opencode
concept: terminal-event-required
commit: ecc4916b5a
files: [packages/opencode/src/session/llm/ai-sdk.ts:23, packages/opencode/src/session/llm/ai-sdk.ts:88-120, packages/llm/src/route/client.ts:279-295, packages/llm/src/route/client.ts:382-391, packages/core/src/session/runner/llm.ts:325-354]
---
[[terminal-event-required]] in [[opencode]].

## Mechanism

### Legacy runtime (AI SDK adapter)
- Unmapped finish reasons become `"unknown"` (`packages/opencode/src/session/llm/ai-sdk.ts:23`) and the loop **continues** on `unknown` (`57fa34f235`).
- A raw finish reason `network_error` is converted to a retryable `ResponseStreamError` instead of a normal finish (`ai-sdk.ts:88-120`, `e0b9e68a68`).
- AI SDK `error` parts fail the stream → session retry.
- No explicit "stream ended without finish" check in the harness; it relies on the AI SDK.

### v2 runtime (`packages/llm` + runner) — gap
- The completeness check "Provider stream ended without a terminal finish event" exists only in `generateWith` (`packages/llm/src/route/client.ts:382-391`). The runner calls `llm.stream` (`packages/core/src/session/runner/llm.ts:241`), whose `streamPrepared` only truncates at a protocol `terminal` and calls `onHalt` (`client.ts:279-295`).
- Per protocol on a cut stream:
  - Anthropic: no `terminal`, no `onHalt`; finish only on `message_delta` (`packages/llm/src/protocols/anthropic-messages.ts:783-795`).
  - Gemini: `onHalt` emits finish only if `finishReason || usage` was seen (`protocols/gemini.ts:379-397,496`).
  - Bedrock: a `metadata` frame without `messageStop` fabricates `"stop"` (`protocols/bedrock-converse.ts:585-587,618-629`).
  - OpenAI Chat: tool-call JSON finalized only at `finish_reason`, so a cut stream silently drops partial tool calls (`protocols/openai-chat.ts:444-447,462-470`).
  - OpenAI Responses: `terminal` = `response.completed|incomplete|failed` (`protocols/openai-responses.ts:613`); earlier socket close → no finish.
- Runner: no `step-finish` → `stepSettlement` undefined → no `Step.Ended`, assistant not failed (`llm.ts:325-346`), `needsContinuation` false → **a truncated text answer ends the drain as success**. No runner test covers it (`packages/core/test/session-runner.test.ts` "terminal" tests cover provider errors only).

## Constants
| name | value | path:line |
|---|---|---|
| Responses `TERMINAL_TYPES` | `response.completed`, `response.incomplete`, `response.failed` | `packages/llm/src/protocols/openai-responses.ts:613` |

## Evolution
- 2026-05-08 `5bb7b23440` native LLM core (generate-only completeness check).
- 2026-08-21 `e0b9e68a68` legacy: raw `network_error` finish → retryable stream error.
- 2026-08-21 `57fa34f235` legacy: `unknown` finish continues.

## Quirks / drift
- Legacy and v2 disagree: legacy treats truncation shapes as retryable; v2 accepts them silently (latent, confirmed by reading code).

Failures: [[truncated-stream-accepted-as-success]] · [[stop-reason-mapping-gaps]] · [[streamed-tool-call-fragmentation]].

Contrast: [[pi--terminal-event-required|pi]] throws in every adapter on a missing terminal; opencode v2 checks only in `generate`.
