---
type: concept
stage: tools
tier: candidate
aliases: [bash tool, powershell tool, bash.ts, bash-executor, OutputAccumulator, EXIT_STDIO_GRACE_MS, bounded-streaming-accumulator, exit-stdio-idle-grace, optional-tool-timeout, session-env-injection, output-window-skips, windowed-exec-output, workdir, OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS, shell.env]
harnesses: [pi, opencode]
---
Mechanics of the model's shell tool: shell selection, spawning, streaming output into bounded memory, timeout policy, deciding when a command is "done", exit-code semantics, environment injection, and cleanup of descendants.

## Why
- "Process exited" ≠ "output finished" ≠ "pipes closed": waiting on the wrong one hangs forever or truncates output ([[bash-descendants-hang-or-lose-output]]).
- Ambiguous terminations (signals) reported as success mislead the model ([[signal-killed-command-reported-success]]).
- Unbounded buffering or per-chunk re-concatenation is quadratic and miscounts lines; chunked UTF-8 decoding corrupts output ([[bash-output-integrity]]).
- Spawn errors (missing cwd/shell) escaping as exceptions crash the session ([[bash-spawn-errors-crash-session]]).
- Model-supplied timeouts can be nonsensical ([[bash-timeout-clamped-to-immediate]]); Windows needs separate shell/kill/encoding paths ([[windows-process-tree-and-shells]]).

## Design space
- Default timeout (fixed, e.g. 30 s — pi v0) vs **none, model-optional, validated** (pi; [[no-bash-default-timeout]]).
- Background jobs / `run_in_background` vs foreground-only (pi; [[no-background-bash]] — "use tmux").
- Persistent shell session (cwd/env survive) vs fresh `bash -c` per call (pi).
- Truncation direction: tail for shell (errors at end) — pi — with spill file for full output → [[tool-output-truncation]], [[tool-output-spill]].
- Bounded streaming accumulator with global line/byte counts (pi `OutputAccumulator`) vs buffer-all.
- Producer-side windowing that drops output the caller won't keep, reporting exact skip counts (pi durable/remote env).
- Completion: resolve on `close` vs on `exit` + idle-pipe grace re-armed per chunk (pi).
- Non-zero exit as thrown error (pi durable) vs error result carrying output + structured data (pi coding-agent).
- Strip ANSI/binary for the model (absent in pi; only for display) vs raw.
- Inject session metadata as env vars (pi `PI_*`) → [[env-vars-as-context]].
- Kill process group on abort/timeout/shutdown → [[process-tree-kill]]; pluggable exec backend → [[pluggable-tool-backends]].
- `workdir` parameter instead of `cd &&` (opencode).
- Per-shell templated descriptions for bash / pwsh / PowerShell 5.1 / cmd (opencode).
- Default timeout with no max, trusting the model (opencode legacy) vs schema-capped (opencode v2 10 min).
- Honest authority statement instead of token-scanning containment (opencode v2) → [[no-sandbox]].
- Parse into sub-commands for permission checks before execution → [[shell-command-permission-parsing]].

## Implementations
- [[pi--shell-execution|pi]] — `bash(command, timeout?)` via `bash -c`, detached process group, stdin ignored, tail-truncated 2000 lines/50KB + private spill file, 100 ms idle exit grace, 128+signal exit codes, `PI_*` env; `powershell` sibling on Windows.
- [[opencode--shell-execution|opencode]] — fresh process per call, `stdin: ignore`, process group, tree-sitter permission parse, 2-min default / no max (v2: 10-min cap), tail + spill, per-shell templated description; background mode removed in v2.

## Failures
- [[tool-description-drifts-from-implementation]]
- [[bash-spawn-errors-crash-session]]
- [[signal-killed-command-reported-success]]
- [[bash-descendants-hang-or-lose-output]]
- [[bash-output-integrity]]
- [[bash-timeout-clamped-to-immediate]]
- [[windows-process-tree-and-shells]]
- [[tools-ignore-session-cwd]]
- [[shell-waits-on-inherited-stdin]]
- [[shell-cd-chaining]]
- [[model-truncates-command-output]]
- [[unrequested-git-commits]]
- [[amend-after-failed-commit]]
- [[background-job-without-observation-tool]]

## Tradeoffs
- [[bash-timeout-default-vs-none]]

## Related
[[process-tree-kill]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[env-vars-as-context]] · [[pluggable-tool-backends]] · [[tool-only-isolation]] · [[structured-tool-output]] · [[abort-propagation]] · [[remote-execution-env]]
