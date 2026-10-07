---
type: implementation
harness: opencode
concept: tool-schema-lowering
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:1492-1573, packages/opencode/src/provider/transform.ts:1575-1716, packages/opencode/src/session/tools.ts:98, packages/llm/src/protocols/utils/tool-schema.ts:1-66, packages/llm/src/protocols/utils/gemini-tool-schema.ts:1-99]
---
[[tool-schema-lowering]] in [[opencode]].

## Mechanism

### Legacy runtime — `ProviderTransform.schema(model, schema)`
- Applied to built-in tools, MCP tools and MCP resource tools before they reach the SDK (`packages/opencode/src/session/tools.ts:98`; `packages/opencode/src/provider/transform.ts:1575`).
- **OpenAI / Azure** (`sanitizeOpenAISchema`, "Mirrors Codex's Rust JSON schema compatibility lowering"): boolean schemas → `{type:"string"}`, `const` → `enum`, keep only known keywords, infer missing `type` from keywords (MCP schemas), objects get `properties: {}`, arrays get `items: {type:"string"}` (`transform.ts:1492-1573`). Paired with `strict: false` ([[opencode--constrained-tool-sampling|constrained-tool-sampling]]).
- **Moonshot / Kimi** (`sanitizeMoonshot`): `$ref` nodes stripped of siblings; tuple `items` → first schema (`transform.ts:1600-1611`).
- **Gemini** (`sanitizeGemini`): integer enums → string enums, type arrays → `anyOf` + `nullable`, `required` filtered to existing properties, empty array items typed string, `properties`/`required` removed on non-objects, no siblings injected next to `anyOf/oneOf/allOf` (`transform.ts:1644-1711`).

### v2 runtime — `ToolSchemaProjection` per model family
- OpenAI: root `anyOf` flattened into one object (`additionalProperties: false`), root `type: "object"` forced, `{type:"null"}` variants removed; applied unconditionally for OpenAI Chat and Responses (`packages/llm/src/protocols/utils/tool-schema.ts:51-66`).
- Moonshot: tuple `items`/`prefixItems` → `anyOf`, `unevaluatedItems` dropped (`tool-schema.ts:20-49`).
- Gemini: numeric enums stringified, `required` filtered, `items` default string, type arrays → first non-null + `nullable`, `const` → `enum`, empty object schemas dropped (`packages/llm/src/protocols/utils/gemini-tool-schema.ts:1-99`).
- Anthropic: `modelCompatibility` only (`packages/llm/src/protocols/anthropic-messages.ts:518-522`).

## Constants
none.

## Evolution
- 2025-06-17 `ee91f31313` Gemini integer enums; 2025-11-26 `ee946d8128` MCP schemas for Gemini; 2025-12-01 `0331931f56` more invalid cases; 2025-12-21 `d89b567b47` MCP arrays without `items`.
- 2026-02-03 `3741516fe3` nested array items, `3adeed8f97` properties/required on non-objects; 2026-03-03 `7e3e85ba59` no sibling injection next to combiners; 2026-06-11 `2e71292f2f` type arrays with null → `anyOf`.
- 2026-05-09 `79d6b10d7c` MCP `outputSchema` with unresolved `$ref` crashed listing → tolerant.
- 2026-06-04 `f011d77128` v2 OpenAI function schemas normalized; 2026-06-16 `213ff3f2d7` Codex lowering port for OpenAI MCP schemas, `25cb2be619` MCP tools without `properties` default `{}`.

## Quirks / drift
- Third-party (MCP) schemas are the main source of rejections; lowering happens at the provider boundary because upstream validity cannot be trusted.
- Flattening a root union in v2 loses variant discrimination (inference) → [[tool-arg-coercion-breaks-unions]] risk.

Failures: [[strict-tool-schema-rejections]].

Contrast: pi lowers only for the legacy Google `parameters` path (`sanitizeForOpenApi`, see [[pi--constrained-tool-sampling|pi]]) and otherwise prefers strict sampling; opencode lowers per family for every provider.
