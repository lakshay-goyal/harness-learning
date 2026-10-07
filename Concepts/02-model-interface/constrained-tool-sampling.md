---
type: concept
stage: tool-design
tier: candidate
aliases: [constrainedSampling, makeStrictJsonSchema, CODEMODE_SOURCE_GRAMMAR, strict-schema-fallback, strict-schema-transformation, grammar-tool-json-bridge, "strict: prefer", "strict: require", resolveJsonSchemaStrictSampling, VALIDATED, StructuredOutput tool, STRUCTURED_OUTPUT_SYSTEM_PROMPT]
harnesses: [pi, opencode]
---
Ask the provider to enforce tool-argument structure while decoding, through a strict JSON schema or a grammar. The schema is rewritten into each provider's supported subset, and the request degrades gracefully when the provider or the schema can't support it.

## Why
- Strict decoding prevents malformed arguments.
- Each provider supports a different schema subset. Anthropic rejects `minimum`/`maximum`/`multipleOf`/most `format`s, unknown OpenAI-compatible servers reject `strict`, and legacy OpenAPI fields reject `$schema`/`$defs`. One unsupported keyword turns every request into a 400 ([[strict-tool-schema-rejections]]).
- Library metadata, such as TypeBox symbol keys, leaks into schemas.

## Design space
- **Opt-in level**
  - None.
  - Per-tool `prefer`, which falls back silently to non-strict.
  - Per-tool `require`, which throws when unsupported. *pi offers both.*
  - Built-in tools default to `prefer` since fcff255b0, after an experimental gate in 7915cdac6.
- **Schema transformation**
  - Send as-is.
  - Strictify: every property required, optional ones become nullable via `anyOf:[x,null]`, `additionalProperties:false`, and rejected composition keywords removed. *pi chose this.*
- **Provider capability**
  - Assume strict works for any OpenAI-compatible server.
  - Default off and let generated metadata opt models in. *pi chose this:* 890f92088.
  - Per-provider keyword veto lists (Anthropic).
- **Grammar**
  - A Lark or regex custom tool, only where supported. The raw grammar output is re-encoded into append-only JSON argument deltas.
  - Otherwise fall back to a normal function tool.
- **Mode coupling**
  - Gemini strict forces the `VALIDATED` calling mode, even over an explicit `auto`. This is a quirk.
- **Strict off + lowering**: force `strict: false` on Responses-family tools for MCP compatibility and rewrite schemas instead (opencode, [[tool-schema-lowering]]).
- **Structured output**: dedicated `StructuredOutput` tool + `toolChoice: required` + system rule (opencode).

## Implementations
- [[pi--constrained-tool-sampling|pi]] — `packages/ai/src/api/constrained-sampling.ts`: `makeStrictJsonSchema`, `resolveJsonSchemaStrictSampling`, and the grammar bridge. Each adapter gates it with compat flags (Anthropic `supportsStrictTools` plus keyword veto, Completions/Responses `supportsStrictMode`, Gemini ≥3, Mistral always on).
- [[opencode--constrained-tool-sampling|opencode]] — `strict: false` (Codex parity); structured output via forced tool.

## Failures
- [[strict-tool-schema-rejections]]

## Related
[[streaming-json-repair]] · [[tool-argument-repair]] · [[code-mode]] · [[model-catalog]] · [[unified-provider-api]]
