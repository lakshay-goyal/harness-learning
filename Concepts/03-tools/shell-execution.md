---
type: concept
stage: tools
tier: candidate
aliases: [bash tool, powershell tool, bash.ts, bash-executor, OutputAccumulator, EXIT_STDIO_GRACE_MS, bounded-streaming-accumulator, exit-stdio-idle-grace, optional-tool-timeout, session-env-injection, output-window-skips, windowed-exec-output, unified exec, exec_command, write_stdin, yield_time_ms, UnifiedExecProcessManager, background terminals]
harnesses: [pi, codex]
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
- **Yield/poll sessions** (✔ codex unified exec): return after `yield_time_ms` (default 10 s, clamp 250–30000 ms, Windows floor 10 s) with partial output + session id; poll / feed stdin with `write_stdin` later — vs run-to-completion (✔ pi).
- Background processes first-class: survive turns and interrupts, 64-slot LRU store protecting 8 MRU, killed by `/stop` or session end (✔ codex) vs none, "use tmux" (✔ pi).
- Kill timeout only in a fallback one-shot mode (✔ codex 10 s, exit 124); default mode never kills on time — the yield model is the timeout policy.
- PTY optional per call (`tty`) vs pipes only (✔ pi; codex default pipes).
- Deterministic child env (NO_COLOR=1, TERM=dumb, C.UTF-8, PAGER/GIT_PAGER/GH_PAGER=cat) instead of ANSI stripping afterwards (✔ codex).
- Login-shell state captured once and sourced per command (✔ codex) → [[shell-environment-snapshot]].
- Collection buffer bounded separately from model budget: 1 MiB head/tail + 10k-token middle elision, no spill (✔ codex) vs tail 50 KB + spill file (✔ pi).
- Approval/sandbox parameters in the shell schema (`sandbox_permissions`, `justification`, `prefix_rule`) (✔ codex) → [[sandbox-escalation-retry]], [[command-rule-policy]].
- Platform-keyed safety rules in the description (✔ codex Windows executors).
- Sniff shell input for the edit DSL: route `apply_patch` heredocs, refuse raw patch bodies (✔ codex) → [[patch-envelope-edit]].
- Post-hoc intent parsing of commands for UI/approvals (✔ codex) → [[shell-command-intent-parsing]]; user-typed commands on a separate path → [[user-shell-escape]].

## Implementations
- [[pi--shell-execution|pi]] — `bash(command, timeout?)` via `bash -c`, detached process group, stdin ignored, tail-truncated 2000 lines/50KB + private spill file, 100 ms idle exit grace, 128+signal exit codes, `PI_*` env; `powershell` sibling on Windows.
- [[codex--shell-execution|codex]] — unified exec `exec_command(cmd, workdir?, tty?, yield_time_ms?, max_output_tokens?, shell?, login?)` + `write_stdin(session_id, chars?)`; 64 persistent processes; deterministic env; 1 MiB head/tail buffer + 10k-token budget; no kill timeout (one-shot fallback 10 s).

## Failures
- [[bash-spawn-errors-crash-session]]
- [[signal-killed-command-reported-success]]
- [[bash-descendants-hang-or-lose-output]]
- [[bash-output-integrity]]
- [[bash-timeout-clamped-to-immediate]]
- [[windows-process-tree-and-shells]]
- [[tools-ignore-session-cwd]]
- [[premature-backgrounding]]
- [[stale-background-process-status]]
- [[patch-body-executed-as-shell]]
- [[windows-destructive-cross-shell]]
- [[unbounded-tool-output-overflows-context]]
- (01) [[interrupt-kills-background-processes]]
- (07) [[harness-credential-leaks-to-tools]]

## Related
[[process-tree-kill]] · [[tool-output-truncation]] · [[tool-output-spill]] · [[env-vars-as-context]] · [[pluggable-tool-backends]] · [[tool-only-isolation]] · [[structured-tool-output]] · [[abort-propagation]] · [[remote-execution-env]] · [[shell-environment-snapshot]] · [[shell-command-intent-parsing]] · [[user-shell-escape]] · [[dedicated-vs-shell-tools]] · [[os-level-sandbox]] · [[elicitation-pause]] · [[no-kill-timeout-in-unified-exec]] · [[removed-legacy-shell-tools]] · [[no-tool-output-spill-file]] · [[no-background-bash]]
