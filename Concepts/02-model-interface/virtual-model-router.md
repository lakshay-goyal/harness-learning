---
type: concept
stage: model-interface
tier: variant
aliases: [registerVirtualModel, jev-router, pi-virtual, ModelRouteReason, pi.virtual-model-state]
harnesses: [pi]
---
A pseudo-model that appears in the model picker. On every request it hands the call to a concrete physical model and effort level, choosing by route reason:
- user turn,
- continuation,
- retry,
- direct side call.

The router can keep state between requests, and that state persists with the session.

## Why
- Plan/implement splits, cost routing and classifier-driven escalation need a model choice per request, but the loop, caching and context accounting expect one selected model.
- Routing has to stay sticky for continuations and retries, or prompt caches and signed reasoning break.
- Routing on the hot path must stay cheap ([[catalog-hot-path-quadratic]]).

## Design space
- **Where routing lives**
  - Client-side cross-provider failover in core. *pi deliberately has no such failover.*
  - A server-side fallback list (see [[server-side-refusal-fallback]]).
  - A plugin-defined virtual model with a `route(request)` callback. *pi chose this,* experimental, 540e174c7.
- **Router inputs**
  - Messages only.
  - Reason, `previous` physical response, `failed` attempt, persisted state, abort signal. *pi chose this.*
- **State persistence**
  - In memory.
  - A session custom entry that follows the session tree and survives compaction. *pi chose this.*
- **Accounting**
  - The context window and cache warming follow the physical model of the latest response. Warming is skipped for routed requests.
  - A virtual id must never shadow a physical id.

## Implementations
- [[pi--virtual-model-router|pi]] — `packages/coding-agent/src/core/virtual-models.ts` plus `registerVirtualModel`. Agent-session routing (`retry`/`user`/`continuation`), `pi.virtual-model-state` entries, and the example `jev-router.ts`.

## Failures
- [[catalog-hot-path-quadratic]]

## Related
[[model-resolution]] · [[model-catalog]] · [[structured-classifier-api]] · [[custom-provider-registration]] · [[auto-retry-backoff]] · [[cache-warming]] · [[signed-reasoning-replay]] · [[extension-event-hooks]]
