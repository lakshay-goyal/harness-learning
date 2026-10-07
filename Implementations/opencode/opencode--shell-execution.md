---
type: implementation
harness: opencode
concept: shell-execution
commit: ecc4916b5a
files: [packages/opencode/src/tool/shell.ts:27, packages/opencode/src/tool/shell.ts:225, packages/opencode/src/tool/shell.ts:293-310, packages/opencode/src/tool/shell.ts:347, packages/opencode/src/tool/shell.ts:418, packages/opencode/src/tool/shell.ts:436-578, packages/opencode/src/tool/shell.ts:603-618, packages/opencode/src/tool/shell/prompt.ts:98-119, packages/opencode/src/tool/shell/prompt.ts:259-261, packages/opencode/src/tool/shell/shell.txt:7-21, packages/core/src/tool/bash.ts:19-21, packages/core/src/tool/bash.ts:109-200]
---
[[shell-execution]] in [[opencode]].

## Mechanism
### Legacy runtime
- Tool id `shell` (shown as bash/pwsh/powershell/cmd). Fresh `ChildProcess` per call with `shell`, `stdin: "ignore"`, `detached` on POSIX (own process group); PowerShell on Windows via `-NoLogo -NoProfile -NonInteractive -Command` (`packages/opencode/src/tool/shell.ts:293-310`) → [[shell-waits-on-inherited-stdin]], [[process-tree-kill]].
- Before execution the command is parsed with tree-sitter (bash or powershell grammar) and each sub-command checked → [[shell-command-permission-parsing]], [[permission-ruleset]].
- Timeout: `params.timeout ?? defaultTimeoutMs`, default `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS ?? 2 min`, **no maximum** (`shell.ts:347,618`). Timeout/abort race the exit; `kill({forceKillAfter: "3 seconds"})` (`:550-554`).
- Expiry text: "shell tool terminated command after exceeding timeout N ms. If this command is expected to take longer and is not waiting for interactive input, retry with a larger timeout value in milliseconds." (`shell.ts:564`).
- Output: rolling in-memory keep `maxBytes × 2`; final model text = `tail(raw, maxLines, maxBytes)`; if cut, full output spilled via `trunc.write` (`shell.ts:439,569-573`) → [[tool-output-truncation]], [[tool-output-spill]]. Empty → "(no output)". UI metadata tail capped at 30 000 chars (`:27`).
- `shell.env` plugin hook injects env vars (`shell.ts:418`) → [[extension-event-hooks]].
- Description rendered per shell with live limits and the configured default timeout (`shell.ts:603`; `packages/opencode/src/tool/shell/prompt.ts:98-119`): dedicated-tool mapping ("Read files: Use Read (NOT cat/head/tail)"), `workdir` instead of `cd &&`, "Do NOT use `head`, `tail`…", git rules (`packages/opencode/src/tool/shell/shell.txt:14,18`), pre-approved `${tmp}` "already been created, already exists" (`shell.txt:7`).
### v2 runtime
- `DEFAULT_TIMEOUT_MS = 2 min`, `MAX_TIMEOUT_MS = 10 min` (schema), `MAX_CAPTURE_BYTES = 1 MiB` (`packages/core/src/tool/bash.ts:19-21`).
- Description: "Execute one shell command string with the host user's filesystem, process, and network authority" (`bash.ts:109`). Absolute paths in the command only yield advisory warnings; spec: "Token scanning cannot honestly provide containment" (`specs/v2/schema-changelog.md:332`) → [[no-sandbox]].
- No model-authored `description` param and no background mode (`d29f5eba92`; `specs/v2/schema-changelog.md:697,702`) → [[background-job-without-observation-tool]].

## Constants
| name | value | path:line |
|---|---|---|
| default timeout | 2 min (env override) | `packages/opencode/src/tool/shell.ts:347` |
| max timeout (legacy) | none | `packages/opencode/src/tool/shell.ts:618` |
| `MAX_TIMEOUT_MS` (v2) | 10 min | `packages/core/src/tool/bash.ts:20` |
| force-kill grace | 3 s | `packages/opencode/src/tool/shell.ts:550` |
| `MAX_METADATA_LENGTH` | 30 000 chars | `packages/opencode/src/tool/shell.ts:27` |
| `MAX_CAPTURE_BYTES` (v2) | 1 MiB | `packages/core/src/tool/bash.ts:21` |

## Evolution
- 2025-05-30 `f3da73553c` 1-min default, 30 000-char output cap; 2025-08-12 `6aa157cfe6` head 1000 lines.
- 2025-08-03 `21c52fd5cb` stuck on interactive commands; 2025-10-12 `b4171aa8e8` `rg` waiting on stdin → `stdin: ignore`.
- 2025-08-31 `029612d8d5` detached + kill process group; 2025-10-16 `fc18fc8a08` "bash hangs & orphans" (SIGTERM → SIGKILL) → [[bash-descendants-hang-or-lose-output]].
- 2025-10-05 `bdf77701cf` timeout message; 2025-10-28 `fc8db6cdf9` timeout must be positive → [[bash-timeout-clamped-to-immediate]].
- 2025-12-06 `75a4dcbce8` 2-min default, 10-min max dropped, `workdir` param → [[shell-cd-chaining]].
- 2026-01-07 `1b82511fbd` spill truncated output to files.
- 2026-04-15 `307251bf3c` tail truncation, bounded memory.
- 2026-04-29 `d4bf70be06` release tree-sitter parse trees (memory leak).
- 2026-05-03 `3f459819ba` shell-aware prompts (bash, pwsh, powershell 5.1, cmd); 2026-05-16 `548648a3d9` description 6287 → 1269 bytes; 2026-05-23 `ba437069e6` advertise configured timeout.
- 2026-06-22 `d29f5eba92` shell `description` input removed (v2 + v1).

## Quirks / drift
- Description still says "Executes a given bash command in a persistent shell session" (`packages/opencode/src/tool/shell/prompt.ts:259`) while every call is a new process; only `workdir` carries cwd → [[tool-description-drifts-from-implementation]].
- Legacy trusts any model timeout (no cap); v2 caps at 10 min.

Contrast: pi has no default timeout and no tree-sitter permission scan → [[pi--shell-execution|pi]], [[no-bash-default-timeout]].
