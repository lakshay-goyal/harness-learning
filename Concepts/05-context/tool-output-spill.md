---
type: concept
stage: context
tier: must-have
aliases: [fullOutputPath, "pi-bash-*.log", "pi-output-<uuid>.log", output-files.ts, writeOutputFile, full_output, output-spill-file, spill, ToolOutputStore, TRUNCATION_DIR, outputPath, outputPaths, "Managed Tool Output File", "tool-output"]
harnesses: [pi, opencode]
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
- Spill failure: kill the command ("Failed to preserve complete shell output") ✔ pi durable NodeExecutionEnv; fail the settlement after the side effect (opencode v2 code) vs record a lossy preview without a path (opencode v2 `CONTEXT.md:194`) — spec and code disagree → [[spill-failure-reports-successful-side-effect-as-failed]].
- **One shared harness directory with retention** (`<data>/tool-output`, 7 days, hourly cleanup) whitelisted for read/grep in every agent's permissions (opencode) vs per-file private temp files (pi).
- **Spill as a delegation trigger**: the hint tells task-capable agents to have the explore subagent grep the file, "Do NOT read the full file yourself" (opencode legacy).
- **Durable record = bounded preview, not the file** (opencode v2); typed `outputPaths` alongside the preview.
- Spill failure: kill the command ("Failed to preserve complete shell output") ✔ pi durable NodeExecutionEnv.
- codex: absent — truncated output is not written to a model-readable file; the full payload survives only in the rollout JSONL, which the model is not pointed at (`codex-rs/core/src/context_manager/history.rs:514-515`); token-budget sessions instead expose server-side history search tools ([[no-tool-output-spill-file]], [[model-requested-context-reset]]).

## Implementations
- [[pi--tool-output-spill|pi]] — `packages/coding-agent/src/utils/output-files.ts` centralized private temp files; bash/powershell/codemode/MCP spill; `OutputAccumulator` flushes buffered chunks into the file on first truncation; durable/env spill thresholds = truncation limits.
- [[opencode--tool-output-spill|opencode]] — legacy `Truncate.output` writes `<data>/tool-output/*`, path in the hint, glob auto-allowed; v2 `ToolOutputStore` exclusive-create `tool_<id>` files with a 7-day global cleanup.

## Failures
- [[model-truncates-command-output]]
- [[bash-output-integrity]]
- [[spill-failure-reports-successful-side-effect-as-failed]]
- [[tool-output-bypasses-truncation]]

## Related
[[tool-output-truncation]] · [[shell-execution]] · [[code-mode]] · [[mcp-integration]] · [[remote-execution-env]] · [[harness-diagnostics-channel]]
