---
type: implementation
harness: opencode
concept: transcript-replay-repair
commit: ecc4916b5a
files: [packages/opencode/src/session/message-v2.ts:258-265, packages/opencode/src/session/message-v2.ts:338-373, packages/opencode/src/provider/transform.ts:101-301, packages/opencode/src/provider/transform.ts:409-445, packages/opencode/src/session/llm/request.ts:159-175, packages/core/src/session/runner/llm.ts:118-138]
---
[[transcript-replay-repair]] in [[opencode]].

## Mechanism

### Legacy runtime — two passes per request
1. **`MessageV2.toModelMessages`** (store → AI SDK messages):
   - Errored assistant messages are skipped, except `AbortedError` messages that have non-reasoning content (`packages/opencode/src/session/message-v2.ts:258-265`).
   - Dangling `pending|running` tool parts → `output-error` "[Tool execution was interrupted]" (Anthropic requires every `tool_use` to have a `tool_result`); interrupted tools with string `metadata.output` → `output-available` with the partial output (`message-v2.ts:338-373`).
   - Empty text separator between signed Anthropic thinking blocks replayed as `" "` (`message-v2.ts:272-289`).
2. **`ProviderTransform.message`** (AI SDK middleware on the final prompt, `packages/opencode/src/provider/transform.ts:465-518`):
   - `unsupportedParts`: file/image parts the model cannot take → text "ERROR: Cannot read <name> (this model does not support <modality> input). Inform the user."; empty base64 image → "ERROR: Image file is empty or corrupted…" (`transform.ts:409-445`).
   - Surrogate sanitization of every text and tool-result string (`transform.ts:101-166`).
   - `@ai-sdk/anthropic`: drop empty strings, empty text, unsigned empty reasoning; drop now-empty messages (`transform.ts:168-195`). Bedrock: same (`transform.ts:196-222`).
   - Tool-call id scrub and Mistral bridge (see [[opencode--tool-call-id-normalization|tool-call-id-normalization]]).
   - DeepSeek: every assistant message gets a (possibly empty) reasoning part; interleaved-field models carry reasoning in `providerOptions.openaiCompatible[field]` even when empty (`transform.ts:303-354`).
- **Copilot `_noop` tool**: when history has tool calls but no tools are enabled (compaction, title), Copilot rejects the request → a placeholder tool "Do not call this tool. It exists only for API compatibility and must never be invoked." is added (`packages/opencode/src/session/llm/request.ts:159-175`).

### v2 runtime
- Before the first turn of every drain, tools still `pending|running` are durably failed "Tool execution interrupted" — never replayed as dangling calls (`packages/core/src/session/runner/llm.ts:118-138`; `specs/v2/schema-changelog.md:631-636`).
- After a successful stream, calls without results → "Provider did not return a tool result" (`llm.ts:349-350`).
- Failed same-model turns: reasoning text kept, provider metadata dropped (`packages/core/src/session/runner/to-llm-message.ts:73`).

## Constants
| name | value | path:line |
|---|---|---|
| interrupted-call placeholder | `[Tool execution was interrupted]` | `packages/opencode/src/session/message-v2.ts:370` |
| v2 interrupted-call failure | `Tool execution interrupted` | `packages/core/src/session/runner/llm.ts:131` |

## Evolution
- 2025-10-16 `d8a15e7bc9` → `d69366b00c` same-day revert: avoid persisting empty thinking/text; 2025-10-15 `1c59530115` revert of a non-whitespace-text fix.
- 2026-01-05 `c285304acf` Anthropic empty messages/reasoning filtered.
- 2026-03-12 `4a2a046d79` Bedrock empty content blocks filtered.
- 2026-03-30 `196a03caff` `_noop` description hardened (models called it); 2026-04-15 `f9d99f044d` Copilot compaction requests; 2026-05-14 `7f7eb2e7f8` LiteLLM workarounds removed (fixed upstream).
- 2026-04-09 `c29392d085` interrupted bash output preserved.
- 2026-05-07 `233fc5b910` signed thinking keeps empty text as `" "`; 2026-06-02 `42173bca4b` → `a763a14d44` same-day revert of signed-thinking reorder.

## Quirks / drift
- Repeated same-day reverts show empty-block rules and signature rules conflict; no provider-agnostic answer.

Failures: [[orphaned-tool-calls-and-results]] · [[failed-turns-replayed]] · [[empty-payload-rejections]] · [[signed-empty-reasoning-dropped]] · [[placeholder-tool-gets-called]] · [[session-switch-leaves-dangling-tool-calls]].

Contrast: [[pi--transcript-replay-repair|pi]] repairs once in `transformMessages` and skips all errored/aborted turns; opencode keeps aborted turns with content and replays interrupted shell output as success.
