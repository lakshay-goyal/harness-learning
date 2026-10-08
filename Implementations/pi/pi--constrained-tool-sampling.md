---
type: implementation
harness: pi
concept: constrained-tool-sampling
commit: b30a6dd77
files: [packages/ai/src/types.ts:714, packages/ai/src/api/constrained-sampling.ts:15, packages/ai/src/api/constrained-sampling.ts:126, packages/ai/src/api/constrained-sampling.ts:170, packages/ai/src/api/constrained-sampling.ts:225, packages/ai/src/api/constrained-sampling.ts:251, packages/ai/src/api/anthropic-messages.ts:1541, packages/ai/src/api/openai-completions.ts:1482, packages/ai/src/api/openai-responses-shared.ts:360, packages/ai/src/api/google-shared.ts:380, packages/ai/src/api/mistral-conversations.ts:763, packages/ai/src/api/bedrock-converse-stream.ts:1140]
---
[[constrained-tool-sampling]] in [[pi]].

## Mechanism
- **Per-tool opt-in** `Tool.constrainedSampling = false | {type:"json_schema", strict:"prefer"|"require"} | {type:"grammar", variants:{openai_lark?, openai_regex?}}` (packages/ai/src/types.ts:714-741; feature 24bace27c #6341).
- Built-in tools default strict-prefer: read/bash/edit/write — not powershell/grep/find/ls (packages/coding-agent/src/core/tools/read.ts:101, bash.ts:260, edit.ts:156, tools/write.ts:57; behind `PI_EXPERIMENTAL` in 7915cdac6, default in fcff255b0); extensions opt out with `constrainedSampling:false`; passed through `wrapToolDefinition` (tool-definition-wrapper.ts:8-30).
- **`makeStrictJsonSchema`** (packages/ai/src/api/constrained-sampling.ts:126-140): clones; rejects `$ref,$defs,definitions,allOf,oneOf,patternProperties,dependentSchemas,dependencies,unevaluatedProperties,propertyNames,contains,prefixItems,not,if,then,else` (:15-32), boolean schemas, tuple items, object/array unions inside anyOf, non-false additionalProperties; makes **all properties required**, wraps originally-optional ones as `anyOf:[prop,{type:null}]`; `additionalProperties:false`; root must be object (:56-124).
- **`resolveJsonSchemaStrictSampling(tool, supportsStrictMode, isUnsupportedKeyword?)`** (:225-249): convertible → `true`; conversion failure → `prefer` returns `undefined` (non-strict fallback), `require` throws; provider unsupported + `require` → throw.
- **Grammar** (:251-298): only with `supportsOpenAIGrammarTools`; Lark preferred over regex; schema must be object with exactly one required string property = grammar input, else throw; unsupported provider → silently normal function tool (:260-262). Streaming bridge `appendGrammarToolInputJsonDelta` re-encodes raw custom-tool text into append-only JSON delta `{"<prop>":"…"}`; non-monotonic / post-close changes throw (:170-200). Codemode uses `CODEMODE_SOURCE_GRAMMAR` Lark so models emit raw JS, not JSON-escaped (packages/codemode/src/source.ts:19-22) → [[code-mode]].

**Adapter degradation table**

| adapter | strict flag source | wire | grammar |
|---|---|---|---|
| Anthropic | `compat.supportsStrictTools` (true only direct anthropic, generate-models.ts:870-871) + keyword veto: `minimum, maximum, exclusiveMinimum, exclusiveMaximum, multipleOf, maxItems, uniqueItems, minContains, maxContains, minProperties, maxProperties`; `minItems` other than 0/1; `format` outside `date-time,time,date,duration,email,hostname,uri,ipv4,ipv6,uuid` (anthropic-messages.ts:1541-1574, 1586; 295cc72b0 #9953) | strict → full strictified schema + `strict:true`; non-strict → only `{type:"object", properties, required}` (:1588-1600) | no |
| Bedrock | `compat.supportsStrictMode` (bedrock-converse-stream.ts:250, 1149) | `toolSpec{…, strict?}` | no |
| OpenAI Completions | `compat.supportsStrictMode !== false`; runtime detect default **false** ("OpenAI compatibility alone does not imply strict"; catalog opts in) (openai-completions.ts:1674-1675; generate-models.ts:781; 890f92088 #9816; Cerebras excluded af7359b90 #9804) | `strict` field omitted if unsupported (:1505-1515) | `{type:"custom", custom:{format:{type:"grammar", grammar:{syntax, definition}}}}` (:1487-1503); replay `{type:"custom"}` (:1361-1372) |
| OpenAI Responses / Azure / Codex | Responses default false, catalog true for openai + cloudflare-ai-gateway (generate-models.ts:864-873); Azure default true (azure-openai-responses.ts:193, 216); Codex `strict:null` default, supportsStrictMode true (openai-codex-responses.ts:537, 548, 575-579) | `parameters` via constrained sampling, `strict` only if supported (openai-responses-shared.ts:360-397) | `{type:"custom", format:{type:"grammar", syntax, definition}}`; grammar tools for gpt-≥5 on openai/openai-codex/azure/github-copilot/opencode/cloudflare-ai-gateway (generate-models.ts:879-898) |
| Google / Vertex | Gemini major ≥3 (`supportsGoogleStrictToolSampling`, google-shared.ts:404-407) | `parametersJsonSchema` (full JSON Schema, 1caadb2e2 #1398); any strict tool → `functionCallingConfig.mode = VALIDATED` unless toolChoice `none`/`any` (overrides explicit `auto`) (:386-436) | no |
| Mistral | always claims support (`resolveJsonSchemaStrictSampling(tool, true)`, mistral-conversations.ts:765) | `strict` boolean; `stripSymbolKeys` removes TypeBox symbol keys (:763-792; 2dddc5ba2 #3361) | no |

- Legacy Google `parameters` (OpenAPI 3.0) path with `sanitizeForOpenApi` stripping `$schema,$id,$anchor,$dynamicAnchor,$vocabulary,$comment,$defs,definitions` (google-shared.ts:345-370; f732f5e85 #3412) — both callers pass `useParameters=false` → dead after Gemini CLI/Antigravity removal (fe66edd94), still unit-tested (google-shared-convert-tools.test.ts:18).
- `tool_choice`: Anthropic string → `{type}`, `{type:"tool",name}` passthrough (anthropic-messages.ts:1273-1279; simple `toolChoice` auto|none e5dde9a76); Bedrock auto/any/tool, `"none"` removes `toolConfig` (bedrock-converse-stream.ts:1140-1175); Google auto/none/any → AUTO/NONE/ANY (google-shared.ts:410-421); Mistral incl. `required` + named (mistral-conversations.ts:909-920); Completions forwards whenever requested even without tools (openai-completions.ts:869-871).
- Validation after sampling still runs: `validateToolArguments` (structuredClone, `normalizeOptionalNulls`, TypeBox `Value.Convert`, compiled validator WeakMap cache) (packages/ai/src/utils/validation.ts:6, 302-350) — strict mode's `null` for optional props normalized back → [[tool-argument-repair]].

## Constants
| name | value | path:line |
|---|---|---|
| strict-rejected keywords (generic) | 16 listed | packages/ai/src/api/constrained-sampling.ts:15-32 |
| Anthropic `format` whitelist | 10 formats | packages/ai/src/api/anthropic-messages.ts:1541-1574 |

## Evolution
- 2026-02-08 `1caadb2e2` Google `parametersJsonSchema`.
- 2026-04-18 `2dddc5ba2` Mistral symbol keys; 2026-04-19 `f732f5e85` meta keys for Cloud Code Assist.
- 2026-07-23 `24bace27c` constrained sampling feature (#6341).
- 2026-08-11 `7915cdac6` strict schema conversion (experimental gate); 2026-09-05 `fcff255b0` strict-prefer default for built-ins.
- 2026-09-20 `af7359b90` Cerebras excluded; 2026-09-21 `890f92088` unknown providers non-strict.
- 2026-09-30 `295cc72b0` Anthropic keyword veto → non-strict fallback.
- Earlier: 0.42.2 boolean strict for LM Studio (#598); 0.51.0 omit strict (#1172) (hashes not mined).

## Evidence commits
24bace27c · 7915cdac6 · fcff255b0 · 295cc72b0 · 890f92088 · af7359b90 · 1caadb2e2 · f732f5e85 · 2dddc5ba2 · e5dde9a76

## Quirks
- Google VALIDATED mode silently overrides an explicit `auto` toolChoice when any tool is strict (google-shared.ts:429-435) — impact on tool-call rate unverified.
- Runtime `detectCompat` vs generator duplication can drift (strict default, `~anthropic/` aliases) (openai-completions.ts:1642, 1675 vs generate-models.ts:749-750, 781).
- Making optional props nullable-required changes what the model sees; `additionalProperties:false` rejected harmless extra edit fields once (a1b336d73 #6278 — 03-tools).

## Failures
- [[strict-tool-schema-rejections]] · [[tool-arg-coercion-breaks-unions]]
