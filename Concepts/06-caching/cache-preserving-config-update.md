---
type: concept
stage: caching
tier: variant
aliases: [ConfigurationUpdate, configuration_update, ReasoningEffortPin, reasoning_effort_pin, record_reasoning_effort_override, supports_reasoning_effort_updates, harness_authored_configuration]
harnesses: [codex]
---
A way to change a sampling setting (e.g. reasoning effort) in the middle of a conversation without changing request parameters. The request-level parameter stays pinned to the value the cached prefix was built with, and a harness-authored "configuration update" item is appended to history instead.

## Why
- On OpenAI-style automatic caching, request parameters are part of the cache identity. Changing `reasoning.effort` mid-window re-bills the whole prefix and invalidates WebSocket delta continuation, because every non-input property must be equal ([[incremental-request-diverges-from-history]]).
- An in-band control item is an injection surface. If client-supplied history can contain it, any client can forge sampling controls. Provenance has to be tracked ("Client-injected history must not be able to forge these controls", `0d502a4230`).
- Compaction and fallback requests must reuse the pinned value. Otherwise the summary request has a different shape from the turn it summarizes ([[compaction-request-shape-mismatch]]).

## Design space
- **Change the request parameter directly.** Simple, but it costs a cache miss on every change. This is the default in most harnesses.
- **Pin the request parameter per context window and append a trusted update item.** The pin is retired by successful compaction. ✔ codex (`codex-rs/core/src/session/reasoning_effort.rs:1-7`)
- **Provenance:**
  - Trust whatever is in history.
  - Accept only harness-authored items, recorded as an envelope metadata flag, and strip client-injected ones. ✔ codex
- **Unsupported models:**
  - Rewrite saved history.
  - Filter the update items from the request copy only. ✔ codex (`codex-rs/core/src/client.rs:902-906`)
- **Value domain:**
  - Any string.
  - Only known backend modes; custom values are excluded. ✔ codex
- **Sibling technique for prompt text:** send a mid-session personality change as a developer message (codex `8b3521ee77`), or, in pi, as section deltas ([[transcript-carried-system-prompt]]).

## Implementations
- [[codex--cache-preserving-config-update|codex]] — `ReasoningEffortPin` plus `ResponseItem::ConfigurationUpdate` with `harness_authored_configuration`. Gated on the feature flag, the OpenAI provider and the catalog's `supports_reasoning_effort_updates`.

## Failures
- [[compaction-request-shape-mismatch]]
- [[incremental-request-diverges-from-history]]

## Related
[[cache-stable-prompt-prefix]] · [[transcript-carried-system-prompt]] · [[thinking-level-abstraction]] · [[mid-turn-settings-switch]] · [[world-state-diff-injection]] · [[session-affinity-cache-routing]] · [[prompt-cache-strategy]]
