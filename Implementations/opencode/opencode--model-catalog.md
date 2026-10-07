---
type: implementation
harness: opencode
concept: model-catalog
commit: ecc4916b5a
files: [packages/core/src/models-dev.ts:136-257, packages/opencode/src/provider/provider.ts:1410-1420, packages/opencode/src/provider/provider.ts:1735-1767]
---
[[model-catalog]] in [[opencode]].

## Mechanism
- Source: **models.dev**, served from `OPENCODE_MODELS_URL` (default `https://models.opencode.ai`) (`packages/core/src/models-dev.ts:160`). The catalog carries limits (`context`, `input`, `output`), cost incl. `experimentalOver200K`, capabilities (`temperature`, `interleaved.field`, modalities), `status` (alpha / deprecated), release date (gates effort tiers).
- Load order (`ModelsDev.Service`): cache file fresh if mtime < 5 min → `OPENCODE_MODELS_PATH` → build-time snapshot `OPENCODE_MODELS_DEV` → fetch under a cross-process file lock (`models-dev.ts:136-199`). Fetch: `Effect.timeout("10 seconds")`, `HttpClient.retryTransient` with `Schedule.exponential(200)` jittered (`models-dev.ts:152-155,180`).
- Background refresh `Schedule.spaced("60 minutes")` (`models-dev.ts:256-257`).
- Provider state filters: `alpha` models hidden unless `OPENCODE_ENABLE_EXPERIMENTAL_MODELS`; `deprecated` removed; providers with no models removed (`packages/opencode/src/provider/provider.ts:1415-1416,1744-1745`).
- v2: plugins register replayable `Catalog.transform()` edits; one rebuild → at most one `Catalog.Event.Updated` (`specs/v2/catalog-config-plugin-lifecycle.md`).

## Constants
| name | value | path:line |
|---|---|---|
| cache file TTL | 5 min | `packages/core/src/models-dev.ts:165` |
| refresh interval | 60 min | `packages/core/src/models-dev.ts:257` |
| fetch attempt timeout | 10 s | `packages/core/src/models-dev.ts:180` |
| unknown context window | `limit.context ?? 0` (0 = no overflow handling) | `packages/opencode/src/provider/provider.ts:1610` |

## Evolution
- 2025-12-03 `6d3fc63658` provider and model system refactor.
- 2026-05-02 `f8738c9002` ModelsDev as an Effect service (hourly refresh, file lock).

## Quirks / drift
- The catalog is operated by the opencode team (models.dev is their project), so the harness and its catalog co-evolve.
- Plugin order constants `{modelsDev:0, env:10, account:20, provider:30, config:40, discovery:50}` exist only in `specs/v2/provider-model.md:350-357` (no such constant in `packages/core/src` at HEAD; spec only).

Failures: [[thinking-config-per-model-drift]] · [[capability-sniffing-misses-opaque-ids]].

Contrast: [[pi--model-catalog|pi]] generates a typed catalog at build time from models.dev and overlays a remote copy every 4 h; opencode reads models.dev at runtime with a 5-min cache and hourly refresh.
