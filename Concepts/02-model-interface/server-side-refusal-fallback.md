---
type: concept
stage: failure-handling
tier: candidate
aliases: [refusal stop reason, fallbacks, allowedFallbackModels, server-side-fallback-2026-07-01, server-side-refusal-fallback, stop_details]
harnesses: [pi]
---
When the provider refuses a request, it retries the request server-side on a fallback model that the client approved in advance. The client must attribute the output and the cost to the model that actually served it, and must treat refusals as errors that carry the provider's explanation.

## Why
- Refusals that look like normal stops end runs silently.
- Refusals that the client does not map to errors lose their explanation ([[stop-reason-mapping-gaps]]).
- Server fallback changes which model served the request. Two things go wrong ([[fallback-model-output-misattributed]]):
  - Pricing at the requested model is wrong.
  - A fallback that starts after output has begun would splice two models' output together.

## Design space
- **Failover location**
  - Client-side failover across providers. *pi has none of this in core.*
  - A server-side fallback list sent per request (`fallbacks`, beta header). *pi chose this:* b03a367a4 configures it, at most 3 entries.
- **Fallback targets**
  - Any model.
  - Generated metadata with local pricing, filtered to models that can take the same effort mode.
- **Mid-output fallback**
  - Splice the outputs.
  - Throw "unsupported mid-output model fallback". *pi chose this.*
  - A fallback block that arrives before any output is skipped transparently.
- **Attribution**
  - Overwrite the model id.
  - Keep the requested `model` (used for replay) and record a separate `responseModel` (used for pricing and display). *pi chose this:* 1283afd0d.
  - The pricing arc took four commits: a6c6f8018 → reverted in 59a71b235 → 4809c2abc → ed867e909.
- **Refusal mapping**
  - `stop`.
  - `error` with `stop_details.explanation`. *pi chose this:* eb1f87fa9.

## Implementations
- [[pi--server-side-refusal-fallback|pi]] — Anthropic `compat.allowedFallbackModels`, the `server-side-fallback-2026-07-01` beta and `fallbacks` param; `message_start` model mismatch sets `responseModel` and fallback cost; refusal maps to an error.

## Failures
- [[fallback-model-output-misattributed]]
- [[stop-reason-mapping-gaps]]

## Related
[[usage-cost-accounting]] · [[unified-provider-api]] · [[virtual-model-router]] · [[cross-provider-handoff]] · [[model-catalog]]
