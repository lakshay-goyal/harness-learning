---
type: concept
stage: tool-design
tier: candidate
aliases: [ToolSpec::Function, ToolSpec::Freeform, "custom tool", ToolSpec::Namespace, ToolSpec::ToolSearch, ToolSpec::WebSearch, ToolPayload::Custom, LoadableToolSpec, freeform tool, namespace tool bundle]
harnesses: [codex]
---
A harness declares tools through several distinct wire types — JSON-schema function, grammar-constrained raw-text ("custom"/freeform) tool, namespace bundles, provider-hosted tools, client-executed hosted search — and routes each call payload kind to the handlers that accept it.

## Why
- Free text with its own syntax (patches, source code) is mangled by JSON-string escaping; a raw-text tool with a grammar avoids the escape layer ([[patch-envelope-edit]], [[code-mode]], [[malformed-tool-json-crashes]]).
- Large tool families (MCP servers, `clock.*`, `web.*`) need grouping and a shared description without flattening names ([[mcp-integration]]).
- Some capabilities execute on the provider (hosted web search) or use provider-native protocol items (tool search) — they are not functions the harness runs ([[web-search-tool]], [[deferred-tool-loading]]).
- A handler receiving the wrong payload kind must fail deterministically (codex: Fatal), or the transcript gets an unpaired/mis-shaped output.

## Design space
- **Single kind: JSON-schema functions only** (pi; grammar used only to constrain one tool's source → [[constrained-tool-sampling]]).
- **Multiple wire kinds serialized straight to the provider's tool JSON** (codex `ToolSpec`: `function`, `namespace`, `tool_search`, `web_search`, `custom`).
- Namespaces may mix function and freeform members (codex since `d4fb78bfc5` 2026-08-04).
- Output schema: sent to the provider vs **kept harness-side only** for script typing (codex `output_schema` is `serde(skip)`) → [[structured-tool-output]].
- Strict flag: enforced vs present-but-unvalidated (codex TODO) → [[constrained-tool-sampling]], [[tool-schema-normalization]].
- Error output keeps the call's wire shape (custom → custom output, tool_search → empty search output) vs one generic error item → [[tool-error-as-result]].
- Provider-specific fallbacks when a backend rejects a kind (codex: Bedrock `custom` rejected → function apply_patch, `0db6811b7c`; [[endpoint-rejects-request-field]]).

## Implementations
- [[codex--tool-wire-kinds|codex]] — `ToolSpec` enum of 5 Responses tool types, `ToolPayload` {Function, ToolSearch, Custom}, handlers declare `matches_kind`, six-level `ToolExposure`.

## Failures
- (02) [[malformed-tool-json-crashes]] · [[grammar-constrained-tool-instability]] · [[endpoint-rejects-request-field]]

## Related
[[constrained-tool-sampling]] · [[patch-envelope-edit]] · [[code-mode]] · [[deferred-tool-loading]] · [[web-search-tool]] · [[mcp-integration]] · [[tool-error-as-result]] · [[structured-tool-output]] · [[tool-schema-normalization]] · [[unified-provider-api]] · [[edit-format]]
