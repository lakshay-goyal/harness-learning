---
type: concept
stage: messages
tier: candidate
aliases: [SystemPrompt.provider, PROMPT_ANTHROPIC, PROMPT_BEAST, PROMPT_CODEX, PROMPT_GPT, PROMPT_ASTRA, PROMPT_GEMINI, PROMPT_KIMI, PROMPT_TRINITY, PROMPT_META, OptimizePlugin, model-specific-system-prompt]
harnesses: [opencode]
---
The harness ships several base system prompts and picks one per request from the model id or provider.

## Why
- Model families respond differently to the same rules: brevity, todo usage, delegation, parallel calls and question-asking all need per-family tuning (see Failures).
- Vendors and model labs contribute prompts tuned to their own model (Meta, Arcee, Moonshot in opencode).
- Cost: N prompts drift independently, copy foreign references from the harnesses they were lifted from ([[borrowed-prompt-foreign-references]]), and must stay consistent with the per-model toolset ([[model-specific-toolset]]).

## Design space
- One prompt for all models (pi, [[minimal-system-prompt]]).
- **Ordered substring chain on the model id, first match wins, full prompt per family** (opencode legacy, 10 prompts).
- **Small shared base + per-family plugin that overrides or appends** (origin/v2 `OptimizePlugin`: GPT/Kimi/Arcee/Meta override, Anthropic appends one paragraph).
- Provider-id fallback when the model id is opaque (opencode Kimi via `moonshotai*` providers).
- Agent prompt replaces the family prompt entirely (opencode legacy and origin/v2: skipped when the agent has its own system).
- Templated model name inside a shared family prompt (opencode `{{MODEL_NAME}}` for Muse).

## Implementations
- [[opencode--per-model-system-prompt|opencode]] — `SystemPrompt.provider(model)`: muse → meta, gpt-4/o1/o3 → beast, gpt-6 → astra, codex → codex, gpt → gpt, gemini- → gemini, claude → anthropic, trinity, kimi → kimi, else default.

## Failures
- [[edit-oldstring-drops-lines]]
- [[borrowed-prompt-foreign-references]]
- [[autonomy-prompt-overreach]]
- [[excessive-permission-questions]]
- [[client-unrenderable-output-format]]
- [[model-cannot-parallel-tool-call]]
- [[over-commenting-code]]
- [[todo-tool-usage-calibration]]
- [[imperative-guideline-over-compliance]]
- [[prompt-names-unavailable-tools]]

## Tradeoffs
- [[single-vs-per-model-system-prompt]]

## Related
[[minimal-system-prompt]] · [[model-specific-toolset]] · [[harness-identity]] · [[provider-identity-shim]] · [[system-prompt-override]] · [[agent-profiles]] · [[ephemeral-reminder-injection]]
