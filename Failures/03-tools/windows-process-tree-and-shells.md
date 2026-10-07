---
type: failure
concepts: [shell-execution, process-tree-kill, path-normalization]
harnesses: [pi]
---
**Symptom** (Windows only)
- `pwsh.exe` produced no stdout/stderr when spawned by the bash/PowerShell tool.
- `$VARS` did not expand when the shell was legacy WSL `C:\Windows\System32\bash.exe`.
- Aborts crashed when `taskkill.exe` was not on PATH; helper console windows popped up from background SDK processes.
- Git-Bash/MSYS drive paths (`/c/...`) resolved against the wrong drive; Windows backslash cwd in the prompt broke model-written bash commands.

**Root cause** — Unix process-group tricks (`detached:true`) break the Windows console host; `-c` args pass through the Windows layer before reaching WSL bash; cleanup depended on PATH; path forms differ between shells.

**Fix · [[pi]]**
- `24dec9fcd` 2026-05-01 (#4013) — no `detached` on Windows; `taskkill /T` doesn't need a group.
- `1287b69fe` 2026-06-19 (#5893) — legacy WSL bash gets `-s` and the command via **stdin** (`packages/coding-agent/src/utils/shell.ts:15-22`).
- `7af2d27dc` 2026-08-26 (#6596) — `System32\taskkill.exe` by absolute path "so cleanup does not depend on PATH" (`shell.ts:187-218`).
- `671798d67` 2026-03-14 (#2080) — prompt cwd backslashes → `/`.
- MSYS/Cygwin/WSL drive mapping in `resolvePath` (`packages/coding-agent/src/utils/paths.ts:67-74`); 0.75.4 (#4699), 0.84.0 (#7064, #7547) (hashes not resolved).
- PowerShell tool (`80e62761f`) prefixes `[Console]::OutputEncoding=UTF8` and runs `-NoProfile -NonInteractive -ExecutionPolicy Bypass` (`shell.ts:122-136`).

**Lesson** — Windows is a second platform for every process, shell-transport and path primitive; don't port Unix process-group semantics.

Related: [[shell-execution]] · [[process-tree-kill]] · [[path-normalization]] · [[bash-descendants-hang-or-lose-output]] · [[pi--shell-execution|pi]]
