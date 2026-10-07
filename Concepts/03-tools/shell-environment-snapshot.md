---
type: concept
stage: tools
tier: candidate
aliases: [ShellSnapshot, shell_snapshots, snapshot_capture_script, maybe_wrap_shell_lc_with_snapshot, Feature::ShellSnapshot, ShellSnapshotV2, login-shell snapshot]
harnesses: [codex]
---
Capture the user's interactive login-shell state (functions, options, aliases, exported variables) once per session into a script, then run every model command in a cheap non-login shell that sources the snapshot first.

## Why
- Model commands need the user's real environment (PATH from nvm/conda/pyenv, aliases, functions) — a bare `bash -c` misses it, so tools "aren't found".
- Running a full login shell (`-lc`) per command re-pays rc-file startup cost on every call and can print noise / prompt.
- rc files can re-introduce secrets the harness deliberately scrubbed, so the snapshot must be filtered and replays must not re-run startup files ([[harness-credential-leaks-to-tools]], [[secret-handling]]).

## Design space
- **Plain non-login `bash -c`; user adds a command prefix** (pi: `shellCommandPrefix` e.g. `shopt -s expand_aliases; source ~/.bash_aliases`) vs **login shell per command** (codex default argv `[shell, "-lc", cmd]`) vs **snapshot once, source per command** (codex `ShellSnapshot`, stable/on).
- Snapshot location: host (codex V1) vs executor side for remote environments (codex `ShellSnapshotV2`, under development).
- Re-application order: harness policy env overrides and runtime vars (`CODEX_THREAD_ID`, PATH prepends) win after sourcing (codex).
- Validity: snapshot only applies when the command cwd equals the snapshot cwd (codex) vs always.
- Secret handling: replace snapshot values with dummies when a credential broker is active; start zsh with `-f` (codex).
- Lifetime: retention window for stale snapshot files (codex 3 days); capture timeout (codex 10 s).
- Shell support: POSIX shells only (codex: PowerShell/cmd unsupported).

## Implementations
- [[codex--shell-environment-snapshot|codex]] — capture script via user shell (zsh/bash/sh), files in `$CODEX_HOME/shell_snapshots`, `shell -lc X` rewritten to `user_shell -c ". SNAPSHOT; exec shell -c X"`, credential-brokerage hardening.

## Failures
- (07) [[harness-credential-leaks-to-tools]]

## Related
[[shell-execution]] · [[env-vars-as-context]] · [[secret-handling]] · [[egress-policy-proxy]] · [[pluggable-tool-backends]]
