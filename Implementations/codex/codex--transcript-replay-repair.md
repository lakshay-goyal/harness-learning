---
type: implementation
harness: codex
concept: transcript-replay-repair
commit: 622e9e3696
files: [codex-rs/core/src/context_manager/history.rs:600-615, codex-rs/core/src/context_manager/history.rs:680-692, codex-rs/core/src/context_manager/history.rs:959-979, codex-rs/core/src/context_manager/history.rs:1028-1051, codex-rs/core/src/context_manager/normalize.rs:19, codex-rs/core/src/context_manager/normalize.rs:52-66, codex-rs/core/src/context_manager/normalize.rs:146-153, codex-rs/core/src/context_manager/normalize.rs:227, codex-rs/core/src/context_manager/normalize.rs:330-420, codex-rs/core/src/util.rs:81-87, codex-rs/core/src/context/unsupported_media.rs:11-19, codex-rs/core/src/context/turn_aborted.rs:22]
---
[[transcript-replay-repair]] in [[codex]].

## Mechanism
- **One choke point on a clone.** `for_prompt` → `normalize_history` runs at prompt-build time on a copy and never mutates stored history (`codex-rs/core/src/context_manager/history.rs:600-615,959-979`). It enforces three documented invariants:
  1. Every call has an output.
  2. Every output has a call, or is a server-executed `tool_search`.
  3. Unsupported media is stripped.
- **Missing outputs are synthesized** as the text `"aborted"`, inserted immediately after the call. Ids are deterministic UUIDv5 values from a fixed namespace, so the repaired prefix is byte-stable across requests (`codex-rs/core/src/context_manager/normalize.rs:19,52-66,146-153`). This covers `FunctionCall`, `ToolSearchCall`, `CustomToolCall` and `LocalShellCall`.
- **Orphan outputs are removed.** `error_or_panic` panics in debug builds and logs `error!` in release (`codex-rs/core/src/util.rs:81-87`), so an invariant violation fails tests loudly but never crashes users.
- **Pair-aware deletion.** When compaction overflow drops the oldest item, its counterpart call or output goes too (`codex-rs/core/src/context_manager/history.rs:680-692`; `codex-rs/core/src/context_manager/normalize.rs:227`).
- **Unsupported media.** Images and audio become "image content omitted because you do not support image input", based on the model's `input_modalities` (`codex-rs/core/src/context/unsupported_media.rs:11-19`; `codex-rs/core/src/context_manager/normalize.rs:330-420`).
- **What counts as an API message.** System-role messages, `CompactionTrigger` and `Other` are excluded from model history (`codex-rs/core/src/context_manager/history.rs:1028-1051`).
- **Interrupt marker.** Aborted turns leave a model-visible `<turn_aborted>` item instead of silently vanishing (`codex-rs/core/src/context/turn_aborted.rs:22`; [[interrupted-turn-invisible-to-model]]).
- **Retry prompts are rebuilt from fresh history**, so completed custom-tool outputs survive an incomplete stream (`a5783f90c9` 2026-04-13).

## Evolution
- `f59978ed3d` 2025-10-23: turn items recorded even on abort ([[turn-items-lost-on-abort]]).
- `1e3cad95c0` 2025-12-15 (#8048): "Do not panic when session contains a tool call without an output". Resuming such a session had panicked ([[session-switch-leaves-dangling-tool-calls]]).
- `b236f1c95d` 2026-01-20 (#9043): `<turn_aborted>` marker.
- `5e01450963` 2026-02-10 (#11349): modality-aware image stripping.
- `a5783f90c9` 2026-04-13: shared cleanup path; retry prompts rebuilt from fresh history ([[orphaned-tool-calls-and-results]]).
- `70a0b1eef8` 2026-07-15: the output-free interrupted prompt is kept in the transcript.

## Versus pi
- Same choice as [[pi--transcript-replay-repair|pi]]: repair at one choke point on every request and add the missing counterpart rather than deleting the call. Text differs: pi "No result provided" with `isError`; codex `"aborted"`.
- Codex adds deterministic ids for synthesized items (cache-stable) and debug-panics on invariant breaks.
- pi skips errored/aborted assistant messages. Codex keeps the completed items and adds an explicit abort marker.

## Failures
- [[orphaned-tool-calls-and-results]]
- [[session-switch-leaves-dangling-tool-calls]]
- [[interrupted-turn-invisible-to-model]]
- [[image-content-poisoning]]
