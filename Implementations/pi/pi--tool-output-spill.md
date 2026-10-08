---
type: implementation
harness: pi
concept: tool-output-spill
commit: b30a6dd77
files: [packages/coding-agent/src/utils/output-files.ts:14, packages/coding-agent/src/core/tools/output-accumulator.ts:239, packages/coding-agent/src/core/tools/bash.ts:286, packages/coding-agent/src/core/bash-executor.ts:78, packages/coding-agent/src/extensions/mcp/tools.ts:135, packages/coding-agent/src/extensions/codemode/execute.ts:390, packages/durable/src/tools/bash.ts:86]
---
[[tool-output-spill]] in [[pi]].

## Mechanism
- **Central module** `packages/coding-agent/src/utils/output-files.ts` ("Every output file is created here, so where they are stored can change in one place", `:1-6`; centralized `d677d0ee7`): path `${tmpdir()}/<prefix>-<16 hex>.<ext>` (`:18-20`); mode `0o600` ("Output can carry private data, so only the user may read the files", `:14-15`); flag `wx` ("never follows a link someone else placed at the path", `:25-26`, `:33`); `writeOutputFile` (one-shot) and `createOutputFileStream` (streamed).
- **Prefixes**: `pi-bash` / `pi-powershell` `.log`, `pi-codemode` `.txt` / image ext, `pi-mcp` `.txt` / resource ext.
- **bash streaming spill** via `OutputAccumulator` (`packages/coding-agent/src/core/tools/output-accumulator.ts`, `6b18cdbac`): spill trigger = raw bytes > maxBytes OR decoded bytes > maxBytes OR lines > maxLines (`:239-243`); on first spill, buffered raw chunks are flushed into the file before continuing (`:245-256`); **raw bytes** (not decoded text) written. Each `onUpdate` snapshot uses `persistIfTruncated: true` so the file exists as soon as truncation happens (`packages/coding-agent/src/core/tools/bash.ts:286-291`, `:333`). `readFullOutput(maxBytes)` for script callers: spilled file > maxBytes → first/last maxBytes/2 with `[... N bytes omitted ...]` (`:149-176`, `1ff5b6fdd`).
- Path surfaced in content `Full output: <path>` and in `details.fullOutputPath` / structured `full_output_path` (`packages/coding-agent/src/core/tools/bash.ts:59`, `343-352`, `394-395`); also on timeout/abort errors (partial output + notice).
- **User `!cmd`** (`packages/coding-agent/src/core/bash-executor.ts:48-152`): rolling buffer 2×50 KB, spill > 50 KB, rendered to the model as `[Output truncated. Full output: <path>]` inside `bashExecution` text (`packages/coding-agent/src/core/messages.ts:94-96`).
- **MCP**: truncated text spilled, `[Full output: path (read it with offset/limit)]`; binary non-image embedded resources saved to temp files (`packages/coding-agent/src/extensions/mcp/tools.ts:124-140`).
- **codemode**: over-budget output spilled to `pi-codemode-*.txt` with `[Full output: path (read with offset/limit)]` (`packages/coding-agent/src/extensions/codemode/execute.ts:373-396`); each `image()` saved to a temp file with a path label (`:333-366`, `d677d0ee7`).

## Constants
| name | value | path:line |
|---|---|---|
| `OUTPUT_FILE_MODE` | `0o600` | `packages/coding-agent/src/utils/output-files.ts:15` |
| random suffix | 8 bytes → 16 hex | `packages/coding-agent/src/utils/output-files.ts:19` |
| spill thresholds | = truncation limits (2000 lines / 50 KB) | `packages/coding-agent/src/core/tools/output-accumulator.ts:239-243` |
| durable spill write `highWaterMark` | 1 MiB (`SPILL_HIGH_WATER_MARK`) | `packages/durable/src/env/node.ts:53` |

## Evolution
- 2025-12-07 `de77cd141` bash truncation + temp file for full output.
- 2026-04-05 `52d16d5a3` (#2852) spill file previously only when bytes > 50 KB → many short lines produced a notice without a file; now any truncation.
- 2026-05-04 `6b18cdbac` (#4145) streaming accumulator flushes buffered raw chunks on spill.
- 2026-05-21 `7dad27e5f` (#4819) TUI shows path once.
- 2026-09-29 `1ff5b6fdd` scripts can read up to 1 MiB; `8562bcf66` MCP/codemode spill.
- 2026-10-04 `d677d0ee7` all output files centralized (0600, `wx`); codemode images to temp files.

## Evidence commits
`de77cd141` `52d16d5a3` `6b18cdbac` `7dad27e5f` `1ff5b6fdd` `8562bcf66` `d677d0ee7` `cdf79797b`

## Quirks
- No cleanup/GC of spill files found (`output-files.ts` has no deletion code; unverified elsewhere) — temp dir grows with long sessions.
- Spill paths are absolute host paths; on remote/SSH backends the path lives on the remote host (durable/env).

## Durable variant (packages/durable)
- Durable bash sets exec `spill{afterBytes, afterLines}` equal to truncation limits; spill path reported as an `info` diagnostic `full_output` ("Full output: path") instead of content (`packages/durable/src/tools/bash.ts:86-108`).
- `NodeExecutionEnv`: output paused while the temp file is created; writer honors backpressure and pauses child stdout/stderr; spill failure kills the command with "Failed to preserve complete shell output" (`packages/durable/src/env/node.ts:741-797`); files `<tmp>/tmp-XXXX/<prefix><uuid><suffix>` (`:1210-1219`).
- Remote env daemon (pi-env): once raw output crosses `spill.afterBytes`/`afterLines` the complete raw output goes to `<tmpdir>/tmp-XXXX/pi-output-<uuid>.log`; path also rides on `timeout`/`aborted` errors (`packages/env/docs/semantics.md:56-57`, `packages/env/daemon/src/exec.rs:451-481`) → [[remote-execution-env]].

## Failures
[[bash-output-integrity]]
