---
type: concept
stage: model-interface
tier: candidate
aliases: [ProviderTransform.temperature, ProviderTransform.topP, ProviderTransform.topK, GEMINI_MODELS_WITH_SAMPLING_DEFAULTS, per-model sampling defaults, vendor-recommended sampling]
harnesses: [opencode]
---
A harness-owned table of temperature, topP and topK for each model family. Values are sent only when they differ from the provider default.

## Why
- Some open-weight and Gemini models degrade (repetition, loops, weak tool use) at the API's default sampling; vendors publish recommended values the API does not apply.
- Sending sampling fields to models that reject them causes 400s ([[endpoint-rejects-request-field]]).

## Design space
- **Where values live**: hardcoded substring table in the harness (opencode) · model catalog metadata · user/agent config only.
- **Omission**: `undefined` = let the provider decide (opencode, e.g. Claude) · always send.
- **Gate**: catalog capability `temperature` (opencode) · try and strip on error.
- **Overrides**: agent `temperature`/`topP`, plugin `chat.params` hook (opencode).
- **Side calls**: own temperature (opencode title agent 0.5).

## Implementations
- [[opencode--sampling-parameter-defaults|opencode]] — `ProviderTransform.temperature/topP/topK` by model-id substring: Gemini 1.0/0.95/64, GLM/MiniMax 1.0, Kimi K2 0.6 (thinking 1.0), Claude omitted.

## Failures
- [[endpoint-rejects-request-field]]

## Related
[[model-catalog]] · [[thinking-level-abstraction]] · [[agent-profiles]] · [[auxiliary-model-calls]]
