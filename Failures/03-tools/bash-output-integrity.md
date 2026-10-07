---
type: failure
concepts: [shell-execution, tool-output-truncation, tool-output-spill]
harnesses: [pi]
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

**Lesson** — Decode streams with a stateful decoder, count incrementally over the whole stream, and always keep a full-output escape hatch whenever you truncate for the model.

Related: [[shell-execution]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[pi--shell-execution|pi]]
