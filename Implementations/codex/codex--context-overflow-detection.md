---
type: implementation
harness: codex
concept: context-overflow-detection
commit: 622e9e3696
files: [codex-rs/codex-api/src/sse/responses_error.rs:48, codex-rs/protocol/src/error.rs:414, codex-rs/core/src/session/turn.rs:1687-1691, codex-rs/core/src/session/turn.rs:758-787, codex-rs/core/src/session/turn.rs:1318, codex-rs/core/src/compact.rs:330-341, codex-rs/codex-api/src/sse/responses.rs:418-433, codex-rs/protocol/src/openai_models.rs:527-538]
---
[[context-overflow-detection]] in [[codex]].

## Mechanism
- **Signal: structured error code only.** `context_length_exceeded` in `response.failed` → `ContextWindowExceeded` (`codex-rs/codex-api/src/sse/responses_error.rs:48`).
  - No message-regex table.
  - No silent-overflow check from usage.
  - No length-stop heuristic.
- **Terminal for retry.** `ContextWindowExceeded` returns `None` from `retry_delay` (`codex-rs/protocol/src/error.rs:414`).
- **On overflow during sampling:**
  - The turn calls `sess.set_total_tokens_full`, pinning usage to the full window, and fails (`codex-rs/core/src/session/turn.rs:1687-1691`).
  - The *next* turn's pre-sampling check compacts first (`codex-rs/core/src/session/turn.rs:1318`).
  - There is no in-turn compact-and-retry for normal sessions ([[no-in-turn-overflow-retry]]).
  - Only the Guardian reviewer path retries once after a summarizing compaction (`codex-rs/core/src/session/turn.rs:758-787`).
- **Overflow inside local compaction** drops the oldest history item and retries ("Trim from the beginning to preserve cache"). If only one item remains, usage is pinned to full and the compaction fails (`codex-rs/core/src/compact.rs:330-341`).
- **Length stops are not overflow.** `response.incomplete` with a reason other than `interrupted` or `content_filter` becomes a retryable Stream error (`codex-rs/codex-api/src/sse/responses.rs:418-433`).
- **Prevention over detection.** Auto-compaction at `min(configured, 90% of window)` (`codex-rs/protocol/src/openai_models.rs:527-538`), with an effective window of 95% ([[codex--model-catalog|codex catalog]]).

## Evolution
- `90ef94d3b3` 2025-10-04 (#4675): "Surface context window error to the client".
- `049a61bcfc` 2025-10-20 (#5292): auto-compact at about 90%.
- `40de788c4d` 2026-02-11: clamp the configured limit to the window ([[auto-compact-threshold-exceeds-window]]).
- `15e79f3c26` 2026-05-11 (#22141): moved overflow handling into the sampling retry loop. Reverted the same day by `69f3183a8e` (#22170). Dogfooders had seen re-entry "indefinitely if the compacted retry still did not fit", and a lost image in the rebuilt request.

## Versus pi
- [[pi--context-overflow-detection|pi]] needs a 25-pattern regex catalogue with exclusions, silent-overflow arithmetic and a length-stop rule, because it serves many providers.
- Codex has one provider family with a stable error code, so detection is trivial. Its effort goes into proactive compaction thresholds instead.

## Failures
- [[auto-compact-threshold-exceeds-window]]
- [[tool-output-bypasses-truncation]]
