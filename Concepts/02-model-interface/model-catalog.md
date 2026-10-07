---
type: concept
stage: model-interface
tier: candidate
aliases: [models.generated.ts, generate-models.ts, models.dev, ModelRuntime, remote catalog, models.json, modelOverrides, withRemoteCatalog, generated-model-catalog, layered-model-catalog, remote-catalog-overlay, dynamic-model-refresh, compat-flag-matrix, conservative-capability-defaults, name-heuristic-capability-detection, model-prompt-cache-ttl-metadata, availability-snapshot, custom-model-id-fallback]
harnesses: [pi]
---
Typed metadata for every model: context window, output cap, costs, input modalities and limits, reasoning levels, cache lifetimes, and per-endpoint compatibility flags. It is generated at build time from public catalogs plus curated overrides, then layered at runtime:
1. remote overlay,
2. user config,
3. plugin providers,
4. per-model overrides.

On top sits an availability view that says which providers are authenticated.

## Why
- Capability checks that sniff model-id substrings rot with every release, and they miss opaque ids such as inference-profile ARNs ([[capability-sniffing-misses-opaque-ids]], [[thinking-config-per-model-drift]]).
- Unknown endpoints have to get conservative capabilities, or they reject strict mode or unsupported fields ([[endpoint-rejects-request-field]], [[strict-tool-schema-rejections]]).
- Layers need clear precedence and recency rules:
  - stale cached overlays mask newer bundled data,
  - authoritative remote lists must replace defaults, not merge into them,
  - upstream catalog drift deletes models.

  See [[catalog-layer-precedence-errors]].
- An availability snapshot that is read synchronously races with asynchronous refresh ([[availability-snapshot-races]], [[credential-refresh-on-availability-path]]).
- Catalog lookups sit on the turn hot path ([[catalog-hot-path-quadratic]]).

## Design space
- **Source**
  - Hand-written tables.
  - Live provider `/models` endpoints.
  - A build-time generator from aggregators (models.dev, OpenRouter, Vercel, Radius) plus curated overrides. *pi chose this.*
  - The generator keeps only tool-capable models, writes typed shards, validates them and swaps them in atomically.
- **Capability flags**
  - Runtime sniffing of URL or id.
  - Generated per-model compat metadata with conservative runtime defaults (e.g. strict off). *pi moved here:* 6184307c3, 890f92088.
  - A user escape hatch for opaque ids, e.g. the `AWS_BEDROCK_FORCE_CACHE` env var.
- **Freshness**
  - Static bundle only.
  - A remote overlay with ETag, a 4h freshness window, and use only when newer than the bundled generation time. *pi chose this:* 54fad505b.
- **User layer**
  - Replace.
  - Upsert by id, plus deep-merged `modelOverrides` as the top layer. *pi chose this.*
  - Custom models inherit defaults from the builtin provider.
- **Cache-lifetime metadata**
  - Assume vendor TTLs.
  - Annotate only where verified, e.g. direct Anthropic only, so cache warming never assumes proxies behave the same. *pi chose this.*
- **Availability**
  - Check per call.
  - A synchronous snapshot with generation and sequence counters, plus provisional marking at registration. *pi chose this.*

## Implementations
- [[pi--model-catalog|pi]] — `scripts/generate-models.ts` produces `models.generated.ts` and the typed shards. Coding-agent `ModelRuntime` composes builtin, pi.dev remote overlay, `models.json`, extension providers and `modelOverrides`, and keeps the availability snapshot.

## Failures
- [[capability-sniffing-misses-opaque-ids]]
- [[thinking-config-per-model-drift]]
- [[endpoint-rejects-request-field]]
- [[catalog-layer-precedence-errors]]
- [[availability-snapshot-races]]
- [[catalog-hot-path-quadratic]]
- [[credential-refresh-on-availability-path]]

## Related
[[model-resolution]] · [[custom-provider-registration]] · [[thinking-level-abstraction]] · [[usage-cost-accounting]] · [[cache-warming]] · [[image-normalization]] · [[virtual-model-router]] · [[layered-settings]]
