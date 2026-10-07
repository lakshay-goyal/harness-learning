---
type: implementation
harness: codex
concept: truncated-tool-call-guard
commit: 622e9e3696
files: [codex-rs/core/src/session/turn.rs:2679, codex-rs/core/src/session/turn.rs:3029, codex-rs/codex-api/src/sse/responses.rs:418, codex-rs/core/src/session/turn.rs:3156]
---
[[truncated-tool-call-guard]] in [[codex]].

## Mechanism
- **Structural guard**: tools are dispatched only from `ResponseEvent::OutputItemDone` (complete items) (`codex-rs/core/src/session/turn.rs:2679-2784`); argument deltas (`ToolCallInputDelta`) only feed UI diff consumers (`:3029-3045`). A call cut off mid-stream never becomes an item, so it is never executed.
- **`response.incomplete`** (`codex-rs/codex-api/src/sse/responses.rs:418-454`): reason `content_filter` → ContentFilter error ([[content-filter-retry-without-guidance]]); reason `interrupted` → treated as completed with `end_turn = Some(false)` (forces follow-up); any other reason (e.g. `max_output_tokens`) → `Stream("Incomplete response returned, reason: …")`, RETRYABLE at loop level → [[auto-retry-backoff]].
- **Partial progress kept**: items completed in the failed attempt stay in history, in-flight tools still drained after a stream error (`codex-rs/core/src/session/turn.rs:3156-3165` regardless of outcome); retry rebuilds its prompt from history (`:1641-1649`) — continues rather than replays from scratch.

## Evolution
- 2026-02-12 `fd7f2aedc7` "Handle response.incomplete (#11558)".

## Quirks
- A length stop is retried, not continued — the same request may hit the same ceiling again (no overflow-specific branch; cf. [[length-stop-recovery]]).

## Versus pi
- [[pi--truncated-tool-call-guard]]: pi answers every call of a length-stopped message with "was not executed… Re-issue"; codex never sees unfinalized calls as items (dispatch on item completion) and treats non-interrupted incomplete responses as retryable stream errors.
