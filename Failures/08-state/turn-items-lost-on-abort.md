---
type: failure
concepts: [partial-message-persistence, abort-propagation]
harnesses: [codex]
---
**Symptom** — Turn items (assistant messages, tool calls, tool outputs) were collected in a vector and appended to history only when the turn succeeded, "losing those items on errors including aborting ctrl+c" — the model and resume lost everything the turn had done.

**Root cause** — Persistence batched per turn on success instead of per completed item.

**Fix · [[codex]]**
- `f59978ed3d` 2025-10-23 "Handle cancelling/aborting while processing a turn (#5543)" — tool calls handle cancellation; turn items bubbled up and recorded even on abort (touched `e95abcdf49:codex-rs/core/src/tools/parallel.rs`, `7dfc3a4dc7:codex-rs/core/src/response_processing.rs`).
- Today: items recorded as each `OutputItemDone` arrives (`codex-rs/core/src/stream_events_utils.rs:347`, `:388-394`); tool outputs recorded as they drain (`codex-rs/core/src/session/turn.rs:2473-2478`); aborted tools get synthetic "aborted by user" outputs (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-380`).

**Lesson** — Persist completed items incrementally; never batch a turn's history behind its success.

Related: [[partial-message-persistence]] · [[abort-propagation]] · [[interrupted-turn-invisible-to-model]] · [[codex--partial-message-persistence|codex]] · [[codex--abort-propagation|codex]]
