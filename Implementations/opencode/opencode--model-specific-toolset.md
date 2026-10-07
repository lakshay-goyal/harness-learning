---
type: implementation
harness: opencode
concept: model-specific-toolset
commit: ecc4916b5a
files: [packages/opencode/src/tool/registry.ts:290-303, packages/opencode/src/session/system.ts:28-51, origin/v2:packages/core/src/plugin/optimize.ts:32-62]
---
[[model-specific-toolset]] in [[opencode]].

## Mechanism
### Legacy runtime
- `ToolRegistry.tools(input)` filters per request: `usePatch = modelID.includes("gpt-") && !includes("oss") && !includes("gpt-4")`; `apply_patch` returned only when `usePatch`, `edit`/`write` only when not (`packages/opencode/src/tool/registry.ts:297-300`).
- Same function drops `websearch` unless the provider/flags allow it (`:293-295`) → [[web-tools]].
- Predicate mismatch: tool routing reads `input.modelID`; prompt routing reads `model.api.id` with different substrings (`packages/opencode/src/session/system.ts:28-51`) → [[per-model-system-prompt]].
### origin/v2 branch
- `OpenAIToolsPlugin` / `AnthropicToolsPlugin` `delete tools.grep; delete tools.glob` for gpt/claude ids, but are left out of `Plugins` "intentionally until we can figure out a good ux for displaying heavy grep/glob usage done via shell" (`origin/v2:packages/core/src/plugin/optimize.ts:32-49,62`).
- The plugin curates tools before tool guidance is rendered, so prompt text follows the curated set (`optimize.ts:78-84`) → [[dynamic-tool-guidelines]].

## Constants
| name | value | path:line |
|---|---|---|
| patch-format gate | `gpt-` minus `oss`, `gpt-4` | `packages/opencode/src/tool/registry.ts:297-298` |

## Evolution
- 2025-08-12 `5cc44c872e` todo tools disabled for qwen; 2025-09-08 `1cea8b9e77` re-enabled.
- 2026-01-17 `b7ad6bd839` apply_patch for OpenAI models (edit/write hidden).
- 2026-01-19 `3515b4ff7d` todo tools omitted for openai models; 2026-01-21 `d9f0287d74` added back.

## Quirks / drift
- Prompt and tool routing drift: gpt-oss gets `gpt.txt` ("Always use apply_patch") without the tool → [[prompt-names-unavailable-tools]].

pi has one toolset for all models (no implementation note; [[minimal-default-toolset]] is pi's axis).
