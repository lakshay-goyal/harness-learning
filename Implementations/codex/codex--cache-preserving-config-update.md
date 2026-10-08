---
type: implementation
harness: codex
concept: cache-preserving-config-update
commit: 622e9e3696
files: [codex-rs/core/src/session/reasoning_effort.rs:1-7, codex-rs/core/src/session/reasoning_effort.rs:28-91, codex-rs/core/src/session/reasoning_effort.rs:93-134, codex-rs/core/src/session/reasoning_effort.rs:136-156, codex-rs/core/src/client.rs:556-560, codex-rs/core/src/client.rs:902-906]
---
[[cache-preserving-config-update]] in [[codex]].

## Mechanism
- **Module contract** (`codex-rs/core/src/session/reasoning_effort.rs:1-7`): "Cache-preserving effort updates and the request-effort baseline for a context window. Only trusted harness items establish overrides. … Successful compaction retires the overrides and allows a fresh request baseline. Fixed-effort workers always use their selected request-level effort. Unsupported models use selected request effort without rewriting saved updates."
- **Pin.** `ReasoningEffortPin` holds the request-level `reasoning.effort`, pinned per model slug for the life of a context window. Sampling establishes the pin. Compaction reads it but never mutates it: "Failed compaction and fallback-model lookups must not mutate the live pin." (`codex-rs/core/src/session/reasoning_effort.rs:93-134`).
- **Update item.** When the user changes effort mid-window, `record_reasoning_effort_override` appends `ResponseItem::ConfigurationUpdate { reasoning }` with envelope metadata `harness_authored_configuration: true` (`codex-rs/core/src/session/reasoning_effort.rs:28-91`).
  - It is deduplicated against the latest *trusted* update in history.
  - After a successful compaction (pin state `Compacted`), the new selection becomes the baseline with no update item.
- **Provenance check.** Client-injected configuration updates are stripped or rejected; only harness-authored ones survive replay. `0d502a4230`: "Client-injected history must not be able to forge these controls".
- **Gating** (`reasoning_effort_override_enabled`, `codex-rs/core/src/client.rs:556-560`):
  - the feature `reasoning_effort_override` is on;
  - the provider is OpenAI;
  - the catalog's `supports_reasoning_effort_updates` is true. It defaults to false.

  If the gate is off, saved `ConfigurationUpdate` items are filtered from the **request copy only**, and persisted history is left unchanged (`codex-rs/core/src/client.rs:902-906`).
- **Bounded values.** `persistent` normalizes to `disabled`. Other `Custom(..)` effort values are kept out of durable updates, "so injected items stay bounded to known backend modes" (`codex-rs/core/src/session/reasoning_effort.rs:150-156`).
- **Mid-turn switch integration.** [[mid-turn-settings-switch]] calls `record_reasoning_effort_override` once per sampling step, before the request (`codex-rs/core/src/session/turn.rs:519`).

## Evolution
- `0d502a4230` 2026-09-02 (#42328): "Durable reasoning configuration updates".
- `56a8470aa0` 2026-09-05 (#43110): effort pin plus configuration updates, with tests for "history prefix and cache-key preservation".
- `35d9e4bc4d` 2026-09-08 (#43796): compaction used the selected effort while sampling used the pinned one, and the old pin survived into the new window. This was fixed so compaction replays the pinned effort ([[compaction-request-shape-mismatch]]).
- `78d4d983d3` 2026-09-18: gate on catalog `supports_reasoning_effort_updates`.

## Quirks
- The model is *told* the new effort in-band while the API parameter keeps the old one. Whether the server applies the in-band value is a backend contract; the client only guarantees the prefix stays stable.
- Earlier analogue: personality changes sent as a `<personality_spec>` developer message (`8b3521ee77` 2026-01-22), so the cached system prompt is untouched.

## Failures
- [[compaction-request-shape-mismatch]]
