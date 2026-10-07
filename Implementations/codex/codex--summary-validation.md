---
type: implementation
harness: codex
concept: summary-validation
commit: 622e9e3696
files: [codex-rs/core/src/compact.rs:369, codex-rs/core/src/compact.rs:750, codex-rs/core/src/compact.rs:798, codex-rs/core/src/compact_remote_v2.rs:430, codex-rs/core/src/compact_remote_v2.rs:475]
---
[[summary-validation]] in [[codex]] — partial.

## Mechanism
- **Pre/mid-turn local compaction**: last assistant message taken as the summary; empty → literal "(no summary available)" (`codex-rs/core/src/compact.rs:750-756`). No check for length truncation or tool calls in the summarizer output ([[no-summary-validation-local]]).
- **Post-turn compaction**: empty summary is a hard error "Post-turn compaction completed without an assistant summary" (`codex-rs/core/src/compact.rs:369-379`); outputs buffered and committed only after success — "failures must leave both the live history and persisted rollout intact" (`:798-801`); failure swallowed with a warning, completed turn preserved (`codex-rs/core/src/session/turn.rs:757`).
- **Remote v2**: the server summary is opaque; validation is structural — exactly one `ResponseItem::Compaction` output, 0 or > 1 → `CodexErr::Fatal` (`codex-rs/core/src/compact_remote_v2.rs:430-492`, `:475-480`).
- **Failure policy**: since `ba8b5d9018` (2026-02-06, "Treat compaction failure as failure state (#10927)") a failed auto-compaction stops the turn instead of continuing; pre-turn failures are reported after preserving the incoming prompt (`codex-rs/core/src/compact.rs:250`, `ee93abb690`) → [[compaction-drops-pending-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| empty-summary placeholder | "(no summary available)" | `codex-rs/core/src/compact.rs:750-754` |

## Evolution
- 2026-02-06 `ba8b5d9018` compaction failure = turn failure.
- 2026-09-10 `ee93abb690` pre-turn failure keeps the prompt.
- 2026-09-18 `49e248d4c3` post-turn compaction requires non-empty summary.

## Versus pi
- [[pi--summary-validation]] rejects error/length-stopped and tool-calling summaries and refuses empty ranges (durable also rejects empty text). codex accepts empty summaries mid-task (placeholder) and only validates shape for server summaries — summary quality is delegated to the server for its own models.
