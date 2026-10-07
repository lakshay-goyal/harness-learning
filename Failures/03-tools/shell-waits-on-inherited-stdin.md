---
type: failure
concepts: [shell-execution]
harnesses: [opencode]
---
**Symptom** — A shell tool call hung forever: interactive commands waited for input, and `rg` run without a path read from stdin.

**Root cause** — The child inherited an open stdin from the harness process; any command that reads stdin blocks with no one to answer.

**Fix · [[opencode]]**
- `21c52fd5cb` 2025-08-03 "fix bash tool getting stuck on interactive commands" (`exec` → `spawn`).
- `b4171aa8e8` 2025-10-12 "`rg` hanging forever when run in bash, waiting for stdin (#3103)".
- HEAD: every spawn uses `stdin: "ignore"` (`packages/opencode/src/tool/shell.ts:298,307`); PowerShell also gets `-NonInteractive` (`:295`).
- Timeout text tells the model to retry with a larger timeout only "if this command … is not waiting for interactive input" (`shell.ts:564`).

**Lesson** — Never give agent subprocesses an inheritable stdin.

Related: [[shell-execution]] · [[bash-descendants-hang-or-lose-output]] · [[opencode--shell-execution|opencode]]
