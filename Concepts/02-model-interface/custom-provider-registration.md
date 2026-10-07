---
type: concept
stage: architecture
tier: candidate
aliases: [pi.registerProvider, ProviderConfigInput, provider-extension-registration, extension-provider-registration, unregisterProvider, refreshModels, native Provider]
harnesses: [pi]
---
A plugin API that adds or overrides a model provider at runtime. A registration can supply:
- endpoint and headers,
- auth, including OAuth login and refresh,
- model lists or live discovery,
- a custom streaming implementation that follows the harness's event contract.

## Why
- No harness can ship every gateway, enterprise proxy or new API.
- Without an extension point, users fork the provider layer.
- Re-registration and override semantics must be predictable. Replace semantics silently drop models or credentials ([[provider-reregistration-replaces-config]]).

## Design space
- **Declarative vs code**
  - Declarative config file only (`models.json`).
  - A code-level registration with an optional `streamSimple`. *pi offers both,* plus a native `Provider` object.
- **Re-registration**
  - Replace.
  - Merge only the defined fields, and validate in isolation so a broken re-registration leaves the stored config untouched. *pi chose this:* 39a9784d2.
- **Model list**
  - Static list.
  - `refreshModels`, either publishing through a generation-checked context or returning definitions.
- **Stream contract**
  - Ad hoc.
  - A documented contract: start, balanced block events, one terminal event, aborts honored, payload/response hooks. A test matrix includes handoff, Unicode and refresh cancellation.
- **Overrides**
  - `modelOverrides` and per-model `baseUrl` apply to plugin providers too.
  - Stored credentials satisfy custom providers.

## Implementations
- [[pi--custom-provider-registration|pi]] — `pi.registerProvider` (legacy `ProviderConfigInput` or native `Provider`) and `unregisterProvider`. Provider-composer layering, an OAuth callback adaptor, and lower-level provider hooks (`before_provider_request/headers`, `after_provider_response`, `provider_stream_event`).

## Failures
- [[provider-reregistration-replaces-config]]

## Related
[[unified-provider-api]] · [[model-catalog]] · [[subscription-oauth-auth]] · [[credential-resolution]] · [[extension-event-hooks]] · [[runtime-plugin-loading]] · [[virtual-model-router]]
