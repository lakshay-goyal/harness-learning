---
type: concept
stage: tool-design
tier: variant
aliases: [usePatch, OpenAIToolsPlugin, AnthropicToolsPlugin, "optimize.*.tools"]
harnesses: [opencode]
---
The registry swaps whole tools by model family (e.g. patch-envelope editing for GPT-family models) so the toolset matches each model's training distribution.

## Why
- Models perform best with the tool shapes they were trained on; a foreign shape costs retries or produces hallucinated calls ([[foreign-harness-tool-hallucination]]).
- Offering two tools for the same job (edit and apply_patch) splits behaviour and doubles description tokens.
- Prompt routing and tool routing must agree, or the prompt names tools the request lacks ([[prompt-names-unavailable-tools]]).

## Design space
- **Swap by model-id substring in the registry** (opencode legacy: `gpt-` minus `oss`/`gpt-4` → `apply_patch` only, else `edit`+`write`).
- Plugin hook that deletes tools per model family before guidance is rendered (origin/v2 `OptimizePlugin`, defined but disabled).
- Toggle auxiliary tools per model (opencode history: todo tools off for qwen, then GPT; both reverted) → [[task-list-tool]].
- One toolset for all models + per-model prompt only (pi) vs both axes → [[per-model-system-prompt]].
- Alias foreign tool names to local ones instead of swapping (not seen).

## Implementations
- [[opencode--model-specific-toolset|opencode]] — `ToolRegistry.tools` filters `apply_patch` vs `edit`/`write` by `input.modelID`; origin/v2 has disabled grep/glob removal plugins for GPT and Claude.

## Failures
- [[prompt-names-unavailable-tools]]
- [[todo-tool-usage-calibration]]
- [[alternate-edit-tool-skips-edit-pipeline]]

## Tradeoffs
- [[edit-tool-variants]]
- [[minimal-vs-rich-toolset]]

## Related
[[patch-envelope-edit]] · [[per-model-system-prompt]] · [[minimal-default-toolset]] · [[foreign-harness-tool-hallucination]] · [[search-replace-edit]] · [[deferred-tool-loading]]
