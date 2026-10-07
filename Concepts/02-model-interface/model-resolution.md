---
type: concept
stage: model-interface
tier: candidate
aliases: [resolveCliModel, findInitialModel, enabledModels, Ctrl+P, parseModelPattern, restoreModelFromSession, defaultModelPerProvider, model-reference-resolution, initial-model-selection-cascade, model-cycling-scope, "--models", ":thinking suffix"]
harnesses: [pi]
---
Turn a user string into one concrete model and thinking level. The string can be `provider/id`, a bare id, a fuzzy match, a glob, or any of these with a `:level` suffix. Resolution must:
- break ambiguity in favor of a model the user can actually call (authenticated), or fail loudly;
- pick a startup model through a fallback cascade;
- restore the model a session was using;
- support cycling within a user-defined scope.

## Why
- Ids collide in several ways:
  - Gateway ids contain slashes, e.g. `zai/glm-5`.
  - The same id is offered by several providers.
  - Some ids contain colons, e.g. OpenRouter `:exacto`.

  Naive parsing then picks the wrong, unauthenticated provider or splits ids incorrectly ([[model-reference-ambiguity]]).
- If any step of the fallback cascade picks a model with no credentials, local or custom setups are blocked ([[unusable-default-model-selected]]).

## Design space
- **Matching order**
  - Fuzzy first.
  - Exact canonical match, then provider split, then bare id, then fuzzy (prefer aliases, else the latest dated version). *pi chose this.*
- **Provider inference**
  - Treat the literal id first.
  - Use the prefix as the provider when it is a known provider. *pi chose this:* 7364696ae.
  - Fall back to the raw id on the one authenticated provider: 1b2c32c65.
- **Ambiguity**
  - Catalog order.
  - Prefer the sole authenticated provider, else an error listing the candidates. *pi chose this:* b04faa2da.
- **Suffix syntax**
  - Always split at the colon.
  - Try the full literal first, then strip the last-colon suffix. *pi chose this:* 9a7863fc9.
  - In strict CLI mode an invalid level fails instead of silently resolving elsewhere.
- **Unknown id**
  - Reject.
  - Clone the provider default's metadata under the custom id and warn. *pi chose this.*
- **Startup**
  - CLI, then scope, then saved default (only if authenticated), then per-provider default table, then first available.
- **Scope**
  - All models.
  - Globs over `provider/id` restricted to authenticated models, cycled with a hotkey; the choice is session-scoped unless the user persists it.

## Implementations
- [[pi--model-resolution|pi]] — `packages/coding-agent/src/core/model-resolver.ts`: `resolveCliModel`, `parseModelPattern`, `findInitialModel`, `restoreModelFromSession`, the scoped-models globs, and Ctrl+P cycling in agent-session.

## Failures
- [[model-reference-ambiguity]]
- [[unusable-default-model-selected]]

## Related
[[model-catalog]] · [[thinking-level-abstraction]] · [[virtual-model-router]] · [[credential-resolution]] · [[layered-settings]]
