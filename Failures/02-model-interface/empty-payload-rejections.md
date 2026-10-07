---
type: failure
concepts: [unified-provider-api]
harnesses: [pi]
---
**Symptom**
- `pi --no-tools` failed with a DashScope 400 `"[] is too short - 'tools'"` (#3649/#3650). LiteLLM and Anthropic proxies have the opposite rule: they require `tools` whenever history contains tool calls (#149).
- Image-only user prompts were rejected for their empty text part (#9797). Empty text and thinking blocks were rejected (#344).
- Bedrock rejected blank required text, empty messages and unknown blocks (#4975), and rejected tool args with `""` keys (#7782).
- An empty `anthropic-beta` header caused 400s.
- Compaction sent `tool_choice` without tools, which gateways rejected (#8607).
- Codex rejected an empty `instructions` field (#4184).

**Root cause** Converters emitted empty containers and fields where providers require "absent" or non-empty values, and gateway requirements conflict.

**Fix · [[pi]]**
- `8f67e0016` 2025-12-08: send `tools: []` when history has tool calls (#149). `3e0ee69b5` 2026-04-24: otherwise omit the tools field, in Anthropic and 4 other adapters (`packages/ai/src/api/openai-completions.ts:855-863`, helper `:93-105`; Anthropic `anthropic-messages.ts:1221-1230`) (#3649/#3650).
- `1b6ddca87` 2026-09-21: drop empty text parts (`openai-completions.ts:1261-1290`) (#9797). Filter empty blocks: 0.31.0 (#344).
- `34719d3f4` 2026-06-01: Bedrock `"<empty>"` placeholder for blank user and tool-result content (`packages/ai/src/api/bedrock-converse-stream.ts:120,931-938,985-1007`); 0.78.1 (#4975). `9e2bfc7c4` 2026-05-18: skip unknown content types instead of throwing. Empty assistant messages are skipped (`bedrock-converse-stream.ts:1010-1014`).
- `98145a6c0` 2026-08-11: `sanitizeBedrockDocument` drops empty-string keys, on replayed input only (`bedrock-converse-stream.ts:940-952,1027`) (#7782).
- `b0217ca6f` 2026-04-21: omit an empty `anthropic-beta` header and move to per-tool `eager_input_streaming`.
- `fe37e9f9b` 2026-08-25 silently dropped `tool_choice` with no tools. Reverted by `6b36eb592` 2026-08-26, which removed toolChoice from the compaction callers instead (#8607). At HEAD the completions adapter forwards `tool_choice` whenever requested (`openai-completions.ts:869-871`).
- `15aa31350` 2026-05-05: Codex `instructions` defaults to "You are a helpful assistant." (`packages/ai/src/api/openai-codex-responses.ts:527-564`) (#4184).

**Lesson** Never send empty containers: normalize them to "absent", or to a literal placeholder where a field is required. Fix the caller rather than silently dropping its explicit intent in the adapter.

Related: [[unified-provider-api]] · [[cross-provider-handoff]] · [[placeholder-text-misleads-model]] · [[endpoint-rejects-request-field]] · [[pi--unified-provider-api|pi]]
