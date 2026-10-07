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
- codex: absent. There is no fallback list in `ResponsesApiRequest` (`codex-rs/codex-api/src/common.rs:279-304`). Policy refusals (`cyber_policy`, `bio_policy`, `misalignment_policy_violation`) map to distinct terminal errors with fallback text (`codex-rs/codex-api/src/sse/responses_error.rs:47-99`). A content-filter stop is retried after injecting model-owned guidance ([[content-filter-retry-without-guidance]]).
  - Server-side *reroute* is observed, not requested: the `openai-model` response header becomes `ResponseEvent::ServerModel` (`codex-rs/codex-api/src/sse/responses.rs:33`; `codex-rs/codex-api/src/common.rs:89-91`); a case-insensitive mismatch with the requested slug emits `EventMsg::ModelReroute{from_model, to_model, reason: HighRiskCyberActivity}` + a warning ("…this request was routed to gpt-5.2 as a fallback…"), once per turn (`codex-rs/core/src/session/turn.rs:2892-2904`; `codex-rs/core/src/session/mod.rs:3946-3984`; `02e9006547` 2026-02-16). Sibling safety signals surface as events only: `ModelVerification` (once per turn), `TurnModerationMetadata`, `SafetyBuffering{use_cases, reasons, show_buffering_ui, faster_model}` while output awaits safety review (`codex-rs/core/src/session/turn.rs:2905-2930`; `566f7bf631` 2026-06-21). The requested model stays the recorded model; no pricing/attribution change.

## Implementations
- [[pi--server-side-refusal-fallback|pi]] — Anthropic `compat.allowedFallbackModels`, the `server-side-fallback-2026-07-01` beta and `fallbacks` param; `message_start` model mismatch sets `responseModel` and fallback cost; refusal maps to an error.

## Failures
- [[fallback-model-output-misattributed]]
- [[stop-reason-mapping-gaps]]

## Related
[[usage-cost-accounting]] · [[unified-provider-api]] · [[virtual-model-router]] · [[cross-provider-handoff]] · [[model-catalog]]
