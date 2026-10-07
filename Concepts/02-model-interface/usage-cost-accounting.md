---
type: concept
stage: cost
tier: candidate
aliases: [calculateCost, Usage.cost, cacheWrite1h, applyServiceTierPricing, usage-normalization, request-cost-accounting, service-tier-pricing, parseChunkUsage, early-usage-capture, bill-before-parse]
harnesses: [pi]
---
Normalize every provider's usage report into one disjoint partition (input, output, cacheRead, cacheWrite with a TTL split, reasoning as a subset of output). Then price each message with the per-model rates:
- request-wide context tiers,
- cache-TTL buckets,
- service-tier multipliers,
- the model that actually served the request.

## Why
- Providers report overlapping fields:
  - `promptTokenCount` includes cached tokens.
  - `completion_tokens` includes reasoning.
  - `cached_tokens` sometimes includes writes.
  - Non-standard field names are common.

  Naive summing double-counts ([[usage-double-counting]]).
- Streamed usage arrives as partial or cumulative patches, possibly with fields missing. Mishandling them zeroes counts or loses them when a stream aborts ([[streamed-usage-misread]]).
- Price buckets have to be modeled explicitly:
  - 1h cache writes at 2× input,
  - long-context tiers,
  - flex/priority/fast tiers,
  - fallback models.

  Without them the cost is wrong ([[usage-priced-at-wrong-rate]], [[fallback-model-output-misattributed]]).
- Model identity used for replay must not be what drives pricing ([[model-relabel-breaks-same-model-check]]).
- A call that was billed but failed to parse still cost money ([[billed-call-lost-on-parse-error]]).

## Design space
- **Normalization**
  - Pass through provider fields.
  - A disjoint partition with per-provider fallback chains for cache fields. *pi chose this.*
  - `reasoning` documented as a subset of output.
- **Streaming**
  - Read usage at the end.
  - Capture usage at the first event and overwrite only non-null fields from cumulative deltas. *pi chose this.*
- **Tiers**
  - Marginal per-token tiers.
  - The highest matching input tier applied to the whole request. *pi chose this:* a9ecf301f.
- **Cache TTL buckets**
  - One write rate.
  - A separate `cacheWrite1h` bucket at 2× input. *pi chose this:* 0be5bb6c9.
- **Served model**
  - Price as the requested model.
  - Price as the server-reported fallback model, from configured fallback costs. *pi chose this.*
- **Cost data source**
  - Generated catalog rates in $/M tokens, with hand-curated authoritative prices for some vendors.

## Implementations
- [[pi--usage-cost-accounting|pi]] — `Usage`, plus `calculateCost` in `packages/ai/src/models.ts:1200-1220`. Per-adapter usage parsers, service-tier multipliers and fallback pricing; the coding-agent aggregates by `responseModel`.

## Failures
- [[usage-double-counting]]
- [[streamed-usage-misread]]
- [[usage-priced-at-wrong-rate]]
- [[billed-call-lost-on-parse-error]]
- [[fallback-model-output-misattributed]]
- [[model-relabel-breaks-same-model-check]]

## Related
[[cache-miss-accounting]] · [[cache-warming]] · [[cache-retention-control]] · [[model-catalog]] · [[server-side-refusal-fallback]] · [[errors-as-stream-events]] · [[token-estimation]]
