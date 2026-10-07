---
type: concept
stage: context
tier: candidate
aliases: [fullOutputPath, "pi-bash-*.log", "pi-output-<uuid>.log", output-files.ts, writeOutputFile, full_output, output-spill-file, spill]
harnesses: [pi]
---
When output is truncated for the model, persist the complete raw output to a private temp file and name its path in the result, so the model can page through it with ordinary read/grep tools instead of re-running the command.

## Why
- Re-running a long build/test to see the truncated part is slow and may not reproduce; the file is a free second look.
- Keeps the context budget small while preserving information.
- The spill must trigger on *every* truncation reason (bytes or lines), otherwise the notice points nowhere ([[bash-output-integrity]]).
- Output can contain secrets → private permissions; temp paths are attack surface → exclusive create.

## Design space
- Spill trigger: any truncation ✔ pi (bytes > limit or lines > limit, raw or decoded); only bytes (pi before `52d16d5a3`).
- File naming/location: `${tmpdir}/<prefix>-<16 hex>.<ext>`, mode 0600, `wx` flag ✔ pi coding-agent; durable/env `<tmpdir>/tmp-XXXX/pi-output-<uuid>.log`.
- Raw vs decoded bytes written: raw ✔ pi.
- Path surfaced in content (`Full output: <path>`) ✔ pi coding-agent vs diagnostics block ✔ pi durable vs structured field (`full_output_path` for scripts).
- Remote envs: spill on the remote host, path rides on timeout/abort errors too ✔ pi env daemon.
- Lifecycle: never cleaned up (pi, unverified that no GC exists) vs session-scoped cleanup.
- Spill failure: kill the command ("Failed to preserve complete shell output") ✔ pi durable NodeExecutionEnv.

## Implementations
- [[pi--tool-output-spill|pi]] — `packages/coding-agent/src/utils/output-files.ts` centralized private temp files; bash/powershell/codemode/MCP spill; `OutputAccumulator` flushes buffered chunks into the file on first truncation; durable/env spill thresholds = truncation limits.

## Failures
- [[bash-output-integrity]]

## Related
[[tool-output-truncation]] · [[shell-execution]] · [[code-mode]] · [[mcp-integration]] · [[remote-execution-env]] · [[harness-diagnostics-channel]]
