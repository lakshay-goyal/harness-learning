---
type: failure
concepts: [custom-provider-registration]
harnesses: [pi]
---
**Symptom** — Custom and extension providers lost state or ignored configuration:
- Re-registering a provider with only overrides wiped its models.
- `modelOverrides` were not applied to extension providers.
- Per-model `baseUrl` was ignored.
- Custom providers demanded a redundant `apiKey` even though a stored credential existed.

**Root cause** — Registration used replace semantics, and override, baseUrl and stored-auth layers were wired only for builtin providers.

**Fix · [[pi]]**
- `39a9784d2` 2026-04-24 — re-registration merges defined values over the previous ones and keeps undefined ones ("legacy ModelRegistry contract"). It validates in isolation, so a broken re-registration throws without touching the stored config (#3651) (`packages/coding-agent/src/core/model-runtime.ts:921-942`).
- `ddb8ed0c7` 2026-05-01 — honor registered model base URLs (#4063).
- `ce6a67fc9` 2026-06-23 — stored credentials satisfy custom providers (#5953).
- `c6251a866` 2026-07-09 — `modelOverrides` apply to extension providers (#6367) (`packages/coding-agent/src/core/provider-composer.ts:553-575`).

**Lesson** — Re-registration is a patch, not a replace. Every config layer must apply uniformly to builtin and plugin providers.

Related: [[custom-provider-registration]] · [[pi--custom-provider-registration|pi]] · [[model-catalog]] · [[catalog-layer-precedence-errors]]
