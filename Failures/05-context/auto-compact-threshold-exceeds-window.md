---
type: failure
concepts: [auto-compaction, context-overflow-detection, token-estimation]
harnesses: [codex]
---
**Symptom** — A configured `auto_compact_token_limit` was used verbatim even when larger than the model's context window, so compaction never triggered before the request overflowed.

**Root cause** — User-configured threshold not clamped against the per-model window at use time (and model switches change the window under a fixed config).

**Fix · [[codex]]** — `40de788c4d` 2026-02-11 "Clamp auto-compact limit to context window (#11516)": `auto_compact_token_limit()` = min(configured/catalog, 90% of context window) (`codex-rs/protocol/src/openai_models.rs:527-539`); separately a hard cap at `effective_context_window_percent` (95) forces compaction regardless of scope (`codex-rs/core/src/session/context_window.rs:84-86`). Window overrides are clamped to `max_context_window` (`codex-rs/models-manager/src/model_info.rs:20-31`). In `BodyAfterPrefix` scope the raw config value is still used first (`codex-rs/core/src/session/context_window.rs:72-74`).

**Lesson** — User-configured thresholds must be clamped against per-model limits at use time, not at config load.

Related: [[auto-compaction]] · [[context-overflow-detection]] · [[token-estimation]] · [[codex--token-estimation|codex]]
