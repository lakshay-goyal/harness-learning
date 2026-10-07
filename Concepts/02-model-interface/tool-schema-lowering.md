---
type: concept
stage: model-interface
tier: candidate
aliases: [ProviderTransform.schema, sanitizeGemini, sanitizeOpenAISchema, sanitizeMoonshot, ToolSchemaProjection, GeminiToolSchema, sanitizeForOpenApi, schema dialect projection]
harnesses: [opencode, pi]
---
Rewrite each tool's JSON Schema into the subset a given provider accepts before sending tool definitions.

## Why
- Providers accept different JSON Schema dialects (Gemini OpenAPI subset, OpenAI structured-outputs subset, Moonshot `$ref` rules); one bad tool schema makes **every** request 400 ([[strict-tool-schema-rejections]]).
- MCP and plugin tools bring third-party schemas the harness cannot fix at the source.

## Design space
- **Where**: at the provider boundary per request (opencode legacy `ProviderTransform.schema`, v2 `ToolSchemaProjection`) · once at tool registration.
- **Scope**: every provider family (opencode) · only the legacy Google `parameters` path (pi `sanitizeForOpenApi`).
- **Strategy**: port a reference implementation (opencode mirrors Codex's Rust lowering for OpenAI) · incremental per-incident rules (opencode Gemini, ~10 fixes).
- **Lossy rewrites**: flatten root `anyOf` (opencode v2 OpenAI), integer enum → string, type arrays → `anyOf`/`nullable`, drop unknown keywords.
- **Pairing**: lowering + `strict: false` (opencode) · strict sampling with a strict-schema transform ([[constrained-tool-sampling]], pi).

## Implementations
- [[opencode--tool-schema-lowering|opencode]] — `sanitizeOpenAISchema` (Codex port), `sanitizeMoonshot`, `sanitizeGemini` in legacy; `ToolSchemaProjection.openAI/gemini/moonshot` in `packages/llm`.
- pi — `sanitizeForOpenApi` strips `$schema/$id/$defs…` for legacy Google `parameters`; see [[pi--constrained-tool-sampling|pi]].

## Failures
- [[strict-tool-schema-rejections]]

## Related
[[constrained-tool-sampling]] · [[unified-provider-api]] · [[mcp-integration]] · [[tool-arg-coercion-breaks-unions]]
