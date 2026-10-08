---
type: implementation
harness: pi
concept: streaming-json-repair
commit: b30a6dd77
files: [packages/ai/src/utils/json-parse.ts:27, packages/ai/src/utils/json-parse.ts:104, packages/ai/src/api/anthropic-messages.ts:551, packages/ai/src/api/anthropic-messages.ts:783, packages/ai/src/api/openai-completions.ts:640, packages/ai/src/api/openai-responses-shared.ts:655, packages/ai/src/api/mistral-conversations.ts:745, packages/ai/src/api/anthropic-messages.ts:1534, packages/ai/src/utils/assistant-message-frame.ts:372]
---
[[streaming-json-repair]] in [[pi]].

## Mechanism
- **`parseStreamingJson(text)`** (packages/ai/src/utils/json-parse.ts:104-124): empty → `{}`; try `JSON.parse` + repair; then `partial-json` parse; then partial-parse of repaired; else `{}` — **never throws**. Called on every tool-arg delta to emit `toolcall_delta` with a best-effort object (`partial-json 0.1.7`, packages/ai/package.json:78; 39c626b6c "args always an object, never undefined").
- **`repairJson`** (:27-83): escapes raw control chars inside strings and doubles backslashes before invalid escapes — models emit invalid `\d`, raw newlines in JSON strings. `parseJsonWithRepair` used by Anthropic owned SSE decoder for every event (packages/ai/src/api/anthropic-messages.ts:551) — SDK strict `JSON.parse` hard-failed on such tool deltas (4b926a30a → revert fc9220d2d → reapply e58d631c8, all 2026-04-21, #3175).
- **Per-adapter accumulation**: Anthropic `input_json_delta` appended to scratch `partialJson`, re-parsed per delta (:737-782); Completions `function.arguments` → `partialArgs` per stream index+id (openai-completions.ts:406-407, 499-556, 640-668); Responses `function_call_arguments.delta` re-parse, `.done` computes missing suffix delta when stream skipped deltas (openai-responses-shared.ts:655-671; 2f8019b61 #2745, 151099e17); Mistral `partialArgs` with `toolcall_end` for all calls only at stream end (mistral-conversations.ts:745-760); Google function calls arrive whole → one delta `JSON.stringify(args)` (google-generative-ai.ts:211-219); pi-messages per-index JSON accumulate (pi-messages.ts:180-274); grammar tools re-encoded into append-only JSON (constrained-sampling.ts:170-200) → [[constrained-tool-sampling]].
- **Finalization**: at block stop final parse and **delete `partialJson`** "so replay only carries parsed arguments" (anthropic-messages.ts:783-814; e2b40dfc8 #3078); Responses `output_item.done` final args + strip (openai-responses-shared.ts:711-727); scratch stripped from partials on error path too → [[stream-scratch-state-persisted]].
- **Execution-time strictness** (agent layer): tool calls from `stopReason:"length"` messages are failed with "Re-issue the tool call with complete arguments" instead of executing salvaged args (`failToolCallsFromTruncatedMessage`, packages/agent/src/agent-loop.ts:268, 478; 351efc828 #6285). An intermediate approach in the same PR (strict-parse finalized args, raw text on `ToolCall.malformedArguments`) was reverted before merge — no `malformedArguments` at HEAD (351efc828 message) → [[truncated-tool-call-guard]], [[length-truncated-tool-calls-executed]]. Responses toolUse with unfinished call (scratch still present) → stream error (openai-responses-shared.ts:766-776; 1b2aa0ca0 #9974).
- **Eager tool-input streaming**: Anthropic per-tool `eager_input_streaming:true` when `supportsEagerToolInputStreaming` (default true); false → legacy `fine-grained-tool-streaming-2025-05-14` beta when tools exist (anthropic-messages.ts:1534-1539, 1605; b0217ca6f). Disabled for `github-copilot` claude-haiku-4.5/sonnet-4/sonnet-4.5 and Fireworks (generate-models.ts:272-276, 1685-1691).
- **Frame reducer** reconstructs unfinished tool calls via `parseStreamingJson`; JSON-prefix tolerance for legacy grammar calls (assistant-message-frame.ts:249-253, 372-490) → [[partial-message-persistence]].
- UI side: renderers guard malformed args (1614e95ec #1259).
- Validation coercion after parse (string numbers, nullable unions) in `validateToolArguments` (validation.ts:175-192; 923b9cb9e #786, 2e95584da #7373 fixes #7328, 0d7c81ec9 #2395 skip AJV under CSP) → [[tool-arg-coercion-breaks-unions]].

## Constants
| name | value | path:line |
|---|---|---|
| empty/failed parse result | `{}` | packages/ai/src/utils/json-parse.ts:104-124 |
| partial-json version | 0.1.7 | packages/ai/package.json:78 |

## Evolution
- 2025-09-16 `39c626b6c` partial JSON parsing of streaming tool args.
- 2026-01-24 `151099e17` Responses `arguments.done`.
- 2026-02-05 `1614e95ec` renderer guards (#1259); 2026-02-12 `ed0cfcbda` malformed trailing JSON tolerated in OpenAI streams (#1424).
- 2026-04-02 `2f8019b61` missing Responses suffix delta (#2745).
- 2026-04-14 `e2b40dfc8` strip `partialJson` on finalize (#3078).
- 2026-04-21 `4b926a30a`/`fc9220d2d`/`e58d631c8` owned Anthropic SSE + `repairJson` (#3175); `b0217ca6f` per-tool `eager_input_streaming`.
- 2026-07-07 `351efc828` refuse length-truncated tool calls (agent).

## Evidence commits
39c626b6c · 151099e17 · 1614e95ec · ed0cfcbda · 2f8019b61 · e2b40dfc8 · 4b926a30a · fc9220d2d · e58d631c8 · b0217ca6f · 351efc828 · 1b2aa0ca0

## Quirks
- Repair for preview, strict for execution: the streaming parser's leniency is deliberately not trusted at execution time (agent-loop.ts:268).
- What regression prompted the same-day revert `fc9220d2d` is unverified (the revert added `agent-session-model-switch-thinking.test.ts`).

## Failures
- [[malformed-tool-json-crashes]] · [[stream-scratch-state-persisted]] · [[streamed-tool-call-fragmentation]] · [[tool-arg-coercion-breaks-unions]] · [[length-truncated-tool-calls-executed]]
