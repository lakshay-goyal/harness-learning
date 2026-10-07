---
type: failure
concepts: [shell-execution, tool-output-truncation, tool-output-spill]
harnesses: [pi, codex]
---
**Symptom**
- Binary output (e.g. `curl` of a video) crashed the TUI.
- Multi-byte UTF-8 characters split across chunks came out corrupted (notably over SSH/containers).
- Output truncated by the 2000-**line** limit (many short lines, < 50KB) printed a truncation notice but **no full-output file**.
- Truncation notices reported wrong "lines X-Y of Z" numbers; streaming re-concatenated and re-truncated the whole rolling buffer per chunk (quadratic); a trailing newline counted as an extra line.
- (durable) U+FEFF characters right after a chunk boundary were dropped; tail-retained output depended on progress-commit timing.

**Root cause** — Byte-chunk decoding without a stateful decoder; spill triggered only on bytes; line counts computed relative to the rolling buffer; Node's streaming `TextDecoder` BOM handling drops a U+FEFF after any chunk boundary.

**Fix · [[pi]]**
- `ad42ebf5f` 2025-12-08 — binary output no longer crashes (sanitize for display; `sanitizeBinaryOutput`).
- `6ddfd1be1` 2026-01-04 (#433), `7293d7cb8` 2026-01-10 (#608) — streaming `TextDecoder` in bash executor.
- `52d16d5a3` 2026-04-05 (#2852) — spill the full output on **any** truncation, not only >50KB.
- `6b18cdbac` 2026-05-04 (#4165) — `OutputAccumulator`: streaming decoder, tail of `2×maxBytes`, global incremental line/byte counts (`packages/coding-agent/src/core/tools/output-accumulator.ts:40-256`).
- `f95306781` 2026-05-21 (#4818) — ignore trailing newline as extra line (`truncate.ts:47-56`).
- `a19c09d9b` 2026-10-04 — durable `StreamDecoder` strips U+FEFF only at stream start (`packages/durable/src/env/decode.ts:1-29`); `cd60a5b99` 2026-10-05 output BOM; durable 1.0.3 deterministic tail window (`packages/durable/CHANGELOG.md:74-75`).

**Fix · [[codex]]**
- Symptom in codex: huge command output in monorepos made sessions sluggish then crash ("Commonality: working in monorepo and large projects … Potential high usage of context", issues #8197, #8358, #7585).
- `fb24c47bea` 2025-12-23 "limit output size for exec command in unified exec".
- `6138909d6e` 2026-07-10 "Keep unified exec output collection bounded" — repeated drains had accumulated into an uncapped buffer → `HeadTailBuffer` 1 MiB, half head / half tail, middle reported `... N bytes omitted ...` (`codex-rs/core/src/unified_exec/head_tail_buffer.rs:11-19`; `codex-rs/core/src/unified_exec/mod.rs:80,231-233`); model budget separate (10k tokens, [[tool-output-truncation]]).
- `748d8ac834` 2026-08-21 UI output delta frames bounded to 8192 bytes (`async_watcher.rs:42`); `4891c4e35f` 2026-09-24 output buffers updated atomically so cancellation cannot split a chunk.
- No spill file: truncated middle bytes are not recoverable (contrast pi).

**Lesson** — Decode streams with a stateful decoder, bound collection memory separately from the model budget (keep head and tail), count incrementally over the whole stream, and offer a full-output escape hatch when you truncate.

Related: [[shell-execution]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[pi--shell-execution|pi]] · [[codex--shell-execution|codex]]
