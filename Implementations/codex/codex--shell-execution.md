---
type: implementation
harness: codex
concept: shell-execution
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/shell_spec.rs:20-160, codex-rs/core/src/tools/handlers/shell_spec.rs:197-266, codex-rs/core/src/tools/handlers/shell_spec.rs:339-345, codex-rs/core/src/unified_exec/mod.rs:73-82, codex-rs/core/src/unified_exec/mod.rs:218-233, codex-rs/core/src/unified_exec/process_manager.rs:93-109, codex-rs/core/src/unified_exec/process_manager.rs:900-995, codex-rs/core/src/unified_exec/process_manager.rs:1573-1640, codex-rs/core/src/unified_exec/process_manager.rs:1722-1792, codex-rs/core/src/unified_exec/async_watcher.rs:34-42, codex-rs/core/src/unified_exec/process.rs:39, codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:57-500, codex-rs/core/src/tools/context.rs:483-560, codex-rs/core/src/exec.rs:63-94, codex-rs/core/src/shell.rs:24-50, e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-381]
---
[[shell-execution]] in [[codex]].

"Unified exec": one shell surface, `exec_command` + `write_stdin`, built on a **yield/poll** model — a call waits at most `yield_time_ms`, then returns whatever output arrived plus a session id if the process is still running; the process lives on in a 64-slot store and is polled or fed stdin later. No kill timeout in the default mode. Also the only read/search surface ([[minimal-vs-rich-toolset]]).

Gate: feature `unified_exec` (Stable, default on, `codex-rs/features/src/lib.rs:1036-1041`); legacy top-level `experimental_use_unified_exec_tool` still maps onto it (`codex-rs/core/src/config/managed_features.rs:306-311`).

## Mechanism

### Schema / text (model-facing)
- `exec_command` (`codex-rs/core/src/tools/handlers/shell_spec.rs:30-110`): description "Runs a command in a PTY, returning output or a session ID for ongoing interaction." (+ Windows rules on Windows executors). Params:
  - `cmd` "Shell command to execute." (required) — one string, run by the user's shell.
  - `workdir?` "Working directory for the command. Defaults to the turn cwd."
  - `tty?` "True allocates a PTY for the command; false or omitted uses plain pipes." (only with `Feature::UnifiedExecTty`)
  - `yield_time_ms?` "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms." — Windows: "Maximum time to wait before returning a session ID for a still-running command. Commands that finish sooner return immediately. For ordinary commands, omit this parameter to use the 10000 ms default. Effective range on Windows is 10000-30000 ms." (`:30-34`)
  - `max_output_tokens?` "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
  - `shell?` "Shell binary to launch. Defaults to the user's default shell."; `login?` "True runs the shell with -l/-i semantics; false disables them. Defaults to true."
  - `environment_id?` "Environment id from <environment_context>. Omit to use the primary environment." (multi-environment only)
  - approval params (`:236-266`): `sandbox_permissions` enum `use_default | with_additional_permissions | require_escalated` ("Per-command sandbox override…"), `justification` "User-facing approval question for `require_escalated`; omit otherwise.", `prefix_rule` "Reusable approval prefix for `cmd`, only with `sandbox_permissions: "require_escalated"`; for example ["git", "pull"].", `additional_permissions` (network / file_system) → [[sandbox-escalation-retry]], [[command-rule-policy]], [[model-requested-permissions]].
- `write_stdin(session_id, chars?, yield_time_ms?, max_output_tokens?)` "Writes characters to an existing unified exec session and returns recent output."; `chars` "Bytes to write to stdin. Defaults to empty, which polls without writing."; `yield_time_ms` "Non-empty writes default to 250 ms and cap at 30000 ms; empty polls wait 5000-300000 ms by default." (`:116-160`).
- Windows safety rules appended only when the **executor** is Windows (`windows_shell_guidance`, `:339-345`): no cross-shell destructive pipelines (PowerShell → `cmd /c`), verify resolved absolute targets before recursive delete/move, `Start-Process -WindowStyle Hidden` → [[windows-destructive-cross-shell]].
- `output_schema` for code-mode callers `{chunk_id?, wall_time_seconds, exit_code?, session_id?, original_token_count?, output}` (`:197-228`) → [[structured-tool-output]].

### Shell & environment
- argv `[shell, "-lc"|"-c", cmd]` for zsh/bash/sh; PowerShell `-NoProfile -Command` when not login; cmd `/c` (`codex-rs/core/src/shell.rs:24-50`); wrapped with the login-shell snapshot → [[shell-environment-snapshot]].
- Deterministic env for model commands `UNIFIED_EXEC_ENV`: `NO_COLOR=1, TERM=dumb, LANG/LC_CTYPE/LC_ALL=C.UTF-8, COLORTERM="", PAGER/GIT_PAGER/GH_PAGER=cat, CODEX_CI=1` (`codex-rs/core/src/unified_exec/process_manager.rs:93-104`) — pagers and colors neutralized at the source instead of stripping ANSI afterwards. Plus sandbox env markers (`CODEX_SANDBOX`, `CODEX_SANDBOX_NETWORK_DISABLED`, `CODEX_PERMISSION_PROFILE`, `CODEX_VERSION`, `codex-rs/core/src/exec_env.rs`) → [[env-vars-as-context]]; internal tokens scrubbed ([[harness-credential-leaks-to-tools]]).
- Runs through the tool orchestrator: approval policy, OS sandbox, network proxy ([[tool-call-gate]], [[os-level-sandbox]], [[egress-policy-proxy]]); network denial text "Network access was denied by the Codex sandbox network proxy." with `LATE_NETWORK_DENIAL_GRACE_PERIOD = 100 ms` (`process_manager.rs:105-107`).

### Yield / poll model
- `clamp_yield_time`: Windows floor `WINDOWS_INITIAL_EXEC_YIELD_TIME_FLOOR_MS = 10_000` then clamp to [`MIN_YIELD_TIME_MS = 250`, `MAX_YIELD_TIME_MS = 30_000`] (`codex-rs/core/src/unified_exec/mod.rs:73-77,218-226`).
- Empty `write_stdin` polls wait between `MIN_EMPTY_YIELD_TIME_MS = 5_000` and `max_write_stdin_yield_time_ms` (default `DEFAULT_MAX_BACKGROUND_TERMINAL_TIMEOUT_MS = 300_000`, floor 5000) (`mod.rs:75-78,169-188`) — stops tight polling loops.
- Early return when the process exits before the deadline (`b5dd189067`: "don't have to wait 10s for trivial commands like ls, sed"); early-exit grace `EARLY_EXIT_GRACE_PERIOD = 150 ms` (`codex-rs/core/src/unified_exec/process.rs:39`).
- PTY vs pipes: `tty=true` allocates a PTY, default plain pipes (`577e1fd1b2`); non-tty `write_stdin` accepts only `\u{3}` (sends interrupt), anything else → `StdinClosed`; after a tty write sleeps 100 ms before polling (`process_manager.rs:109,983-995`).
- Elicitation pause: unified exec subscribes to the session's paused flag so user think-time doesn't count against its timers (`process_manager.rs:644`) → [[elicitation-pause]].
- `write_stdin` to a terminal that was granted extra permissions may need reviewer approval; action+reason > `MAX_STDIN_APPROVAL_BYTES = 8_000` → rejected "too large to review safely" rather than executing an unreviewed tail (`process_manager.rs:108,900-935`; `bce96bcb43`) → [[llm-approval-reviewer]].

### Process store (background terminals)
- `MAX_UNIFIED_EXEC_PROCESSES = 64` (soft cap); when full, prune LRU but protect the 8 most recently used, preferring exited ones; never evict a live process because an exited one is briefly locked (`mod.rs:82`; `process_manager.rs:1722-1792`).
- Processes survive turns and **interrupts** (`ba463a9dc7`); killed only by `/stop` (`Op::CleanBackgroundTerminals`, `codex-rs/core/src/session/handlers.rs:501-504`), explicit close, or session shutdown (`codex-rs/core/src/session/handlers.rs:305-312`, `codex-rs/core/src/tasks/mod.rs:906-912`) → [[interrupt-kills-background-processes]]. Interrupt marker tells the model "Any running unified exec processes may still be running in the background." (`codex-rs/core/src/context/turn_aborted.rs:10`).

### Output collection & result
- Collection loop drains a shared buffer into `HeadTailBuffer` (1 MiB `UNIFIED_EXEC_OUTPUT_MAX_BYTES`, half head / half tail, middle reported `... N bytes omitted ...`) until the deadline; after exit waits ≤ `POST_EXIT_CLOSE_WAIT_CAP = 50 ms` for pipe close (`process_manager.rs:1573-1640`; `codex-rs/core/src/unified_exec/head_tail_buffer.rs:11-19`; `mod.rs:80,231-233`); trailing-output grace `TRAILING_OUTPUT_GRACE = 100 ms`; UI delta frames ≤ 8192 bytes (`codex-rs/core/src/unified_exec/async_watcher.rs:34,42`).
- Model budget: `max_output_tokens` (default `DEFAULT_MAX_OUTPUT_TOKENS = 10_000`) capped by the per-model `truncation_policy` (catalog: tokens 10000) — whichever is smaller (`codex-rs/core/src/tools/context.rs:483-491`); middle elision with "Warning: truncated output (original token count: N)" (`context.rs:520`) → [[tool-output-truncation]]. **No spill file** — truncated bytes are gone (no [[tool-output-spill]] equivalent found; [[no-tool-output-spill-file]]).
- Plain-text result header: `Chunk ID: x` / `Wall time: 0.1234 seconds` / `Process exited with code N` | `Process running with session ID N` / `Original token count: N` / `Output:` (`context.rs:524-548`).
- Sandbox denial → normal output (exit code + output, no session id) (`codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:441-466`); other errors → `"exec_command failed: {err:?}"` middle-truncated to `EXEC_COMMAND_REJECTION_MAX_BYTES = 900` (`exec_command.rs:57,467-473`) → [[tool-error-as-result]].
- Abort → synthesized output `"Wall time: X seconds\naborted by user"` (exec_command) / `"aborted by user after Xs"` (others) (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-381`) → [[abort-propagation]].
- `exec_command` with an `apply_patch <<'EOF'` script is intercepted and routed to the patch path; raw patch bodies refused (`exec_command.rs:372`) → [[patch-envelope-edit]].
- Parallel-safe (`supports_parallel_tool_calls = true`, `exec_command.rs:142`) — several commands run concurrently, with no per-file lock ([[parallel-tool-execution]], [[per-file-mutation-queue]]).
- `parse_command` attaches read/search/list intents to begin/end events ([[shell-command-intent-parsing]]).

### One-shot fallback (managed config disables unified exec)
- `ExecCommandHandler::one_shot`: no `tty` / `yield_time_ms` / `session_id` / `write_stdin`; adds `timeout_ms` (default `DEFAULT_EXEC_COMMAND_TIMEOUT_MS = 10_000`); kills on timeout or cancellation; exit code `EXEC_TIMEOUT_EXIT_CODE = 124` (`codex-rs/core/src/exec.rs:63,70`; `exec_command.rs:299-312,478-500`; `b836aecd4d`).

### Legacy `exec.rs` (user `!` commands + sandbox exec)
- Serves [[user-shell-escape]] and sandbox exec (`codex-rs/core/src/tasks/user_shell.rs:15`, `codex-rs/core/src/sandboxing/mod.rs:13`): `SIGKILL_CODE = 9`, `TIMEOUT_CODE = 64`, signal exit `128+n`, cancellation grace 50 ms, read chunk 8192 B, output cap 1 MiB (`DEFAULT_OUTPUT_BYTES_CAP`, `codex-rs/utils/pty/src/lib.rs:25`), ≤ 10_000 output delta events per call, `IO_DRAIN_TIMEOUT_MS = 2_000` after exit "2 s should be plenty for local pipes" (`codex-rs/core/src/exec.rs:63-94`) → [[bash-descendants-hang-or-lose-output]].

### Prompt-side shell rules (per-model instructions / bundled catalog, M5a)
- Prefer `rg` ("(If the `rg` command is not found, then use alternatives.)") and chunked reads — restored by `90d892f4fd` 2025-08-12 after the GPT-5 rewrite `81b148bda2` dropped them ([[prompt-rewrite-drops-load-bearing-lines]]); output-limit literals later removed (`570eb5fe78`, `26d0d822a2`) → [[prompt-states-stale-harness-limits]].
- "Do not use python scripts to attempt to output larger chunks of a file." (`90d892f4fd`; `codex-rs/protocol/src/prompts/base_instructions/default.md:265`) — model evading truncation.
- "The user does not command execution outputs. When asked to show the output of a command (e.g. `git show`), relay the important details" (`916fdc2a37`, typo in original) — model assumed the user sees tool output.
- "Do not chain shell commands with separators like `echo "====";` or `printf '---'`; the output becomes noisy" (`c10f95ddac` 2026-04-24, models.json).
- "Never repurpose `$HOME`, `$home`, or `$CODEX_HOME`." (`d26a9bf671` 2026-07-18, models.json).
- "Avoid performing blocking sleep or wait calls longer than 60 seconds" (`3380969a29` 2026-07-09) → [[wall-clock-tools]].
- Legacy `shell_command`: "Always set the `workdir` param… Do not use `cd` unless absolutely necessary." (`916fdc2a37`, removed `8a40095ea3`).

## Constants
| name | value | path:line |
|---|---|---|
| default yield (exec) | 10_000 ms | `codex-rs/core/src/tools/handlers/shell_spec.rs:33` |
| `MIN_YIELD_TIME_MS` / `MAX_YIELD_TIME_MS` | 250 / 30_000 ms | `codex-rs/core/src/unified_exec/mod.rs:73,77` |
| `WINDOWS_INITIAL_EXEC_YIELD_TIME_FLOOR_MS` | 10_000 ms | `codex-rs/core/src/unified_exec/mod.rs:74` |
| `MIN_EMPTY_YIELD_TIME_MS` | 5_000 ms | `codex-rs/core/src/unified_exec/mod.rs:76` |
| `DEFAULT_MAX_BACKGROUND_TERMINAL_TIMEOUT_MS` (empty-poll max) | 300_000 ms | `codex-rs/core/src/unified_exec/mod.rs:78` |
| non-empty write_stdin default yield | 250 ms | `codex-rs/core/src/tools/handlers/shell_spec.rs:134` |
| `DEFAULT_MAX_OUTPUT_TOKENS` | 10_000 | `codex-rs/core/src/unified_exec/mod.rs:79` |
| `UNIFIED_EXEC_OUTPUT_MAX_BYTES` | 1 MiB (head/tail halves) | `codex-rs/core/src/unified_exec/mod.rs:80` |
| `MAX_UNIFIED_EXEC_PROCESSES` / protected MRU | 64 / 8 | `codex-rs/core/src/unified_exec/mod.rs:82`; `codex-rs/core/src/unified_exec/process_manager.rs:1775` |
| `POST_EXIT_CLOSE_WAIT_CAP` | 50 ms | `codex-rs/core/src/unified_exec/process_manager.rs:1578` |
| `TRAILING_OUTPUT_GRACE` / `EARLY_EXIT_GRACE_PERIOD` | 100 ms / 150 ms | `codex-rs/core/src/unified_exec/async_watcher.rs:34`; `codex-rs/core/src/unified_exec/process.rs:39` |
| `UNIFIED_EXEC_OUTPUT_DELTA_MAX_BYTES` | 8192 | `codex-rs/core/src/unified_exec/async_watcher.rs:42` |
| `MAX_STDIN_APPROVAL_BYTES` | 8_000 | `codex-rs/core/src/unified_exec/process_manager.rs:108` |
| tty post-write wait | 100 ms | `codex-rs/core/src/unified_exec/process_manager.rs:991-995` |
| `EXEC_COMMAND_REJECTION_MAX_BYTES` | 900 | `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:57` |
| one-shot `DEFAULT_EXEC_COMMAND_TIMEOUT_MS` / timeout exit | 10_000 ms / 124 | `codex-rs/core/src/exec.rs:63,70` |
| legacy `IO_DRAIN_TIMEOUT_MS` | 2_000 ms | `codex-rs/core/src/exec.rs:94` |
| `UNIFIED_EXEC_ENV` | 10 vars (NO_COLOR, TERM=dumb, C.UTF-8, pagers=cat, CODEX_CI=1) | `codex-rs/core/src/unified_exec/process_manager.rs:93-104` |

## Evolution
- 2025-04 TS era → Rust one-shot `shell` tool (argv array).
- 2025-09-10 `c09ed74a16` "Unified execution (#3288)" — PTY-backed long-lived sessions (262 commits mention unified exec since).
- 2025-09-14 `916fdc2a37` legacy `shell_command` description "Always set the `workdir` param… Do not use `cd` unless absolutely necessary." (removed with the tool).
- 2025-11-12 `73ed30d7e5` IO drain timeout after exit (grandchild pipe hang, issue #3204).
- 2025-11-13 `0792a7953d` default yield 10 s exec / 250 ms write_stdin. 2025-11-20 `b5dd189067` early exit.
- 2025-12-23 `fb24c47bea` limit exec output size (monorepo crashes #8197/#8358/#7585).
- 2026-01-14 `577e1fd1b2` piped process replaces PTY when not needed; `32b1795ff4` min yield for empty write_stdin ("After evals, 0 impact on performance").
- 2026-02-09 `c2ca51273f` notify instead of grace to close process.
- 2026-02-19 `547f462385` configurable write_stdin max, default 30 s → 5 min; `9719dc502c` "no timeout mode" → reverted same day `928be5f515`.
- 2026-03-15 `ba463a9dc7` background terminals survive interrupt, `/stop` cleans them.
- 2026-03-19 `69750a0b5a` / 2026-04-23 `2e228969be` Windows destructive-command and `-WindowStyle Hidden` rules; 2026-08-28 `5ed294d49d` keyed to executor platform.
- 2026-05-13 `83decfa300` raw `shell`, `local_shell`, `container.exec` removed ("no active use").
- 2026-05-18 `82061660ae` legacy JSON-structured shell output removed. 2026-05-19 `e43a2e297f` stale background poll events.
- 2026-06-15 `e7a9988d1a` Windows yield floor 2000 ms (premature backgrounding: tool calls/turn +20.7 %, tokens/turn +4.1 %, latency/turn +8.3 %, DAU −1.0 %; PowerShell wrapper overhead ~0.7–1.0 s); 2026-07-27 `fd41e813cb` floor raised to 10 s.
- 2026-07-10 `6138909d6e` bounded output collection (repeated drains had grown an uncapped buffer).
- 2026-07-13 `bb947e8e36` telemetry tagged by command category.
- 2026-08-20 `8a40095ea3` / `bce5f2fcfc` `shell_command` removed — unified exec is the only shell surface.
- 2026-08-21 `748d8ac834` bounded output delta frames. 2026-08-27 `bce96bcb43` reject oversized reviewed stdin. 2026-08-28 `b836aecd4d` one-shot fallback.
- 2026-09-23 `d47b9a8c00` early output kept in completion events; 2026-09-24 `4891c4e35f` output buffers updated atomically.

## Quirks
- Description says "Runs a command in a PTY" while the default (`tty` omitted) is plain pipes (`shell_spec.rs:50` vs `:99-103`).
- No wall-clock kill in unified mode: a runaway command keeps a slot until `/stop`, eviction (64-cap LRU) or session end; the "no timeout mode" experiment was reverted, so the yield model *is* the timeout policy ([[no-kill-timeout-in-unified-exec]]).
- Two exec stacks coexist: unified exec (model) and legacy `exec.rs` (user `!`, sandbox exec) with different constants.
- Parallel shell calls may write the same files concurrently — no mutation queue (`862b2122ee` removed `is_mutating` gating).

## Versus pi
pi: run-to-completion `bash -c`, no default timeout, tail-truncated 2000 lines/50 KB + spill file, no background mode ("use tmux") ([[pi--shell-execution]], [[no-background-bash]], [[no-bash-default-timeout]]). codex: yield/poll sessions that persist across turns, head+tail token budget without spill, deterministic env instead of ANSI stripping, login-shell snapshot, approval/sandbox params in the schema.

## Failures
[[bash-descendants-hang-or-lose-output]] · [[bash-output-integrity]] · [[premature-backgrounding]] · [[stale-background-process-status]] · [[patch-body-executed-as-shell]] · [[windows-destructive-cross-shell]] · [[tool-output-bypasses-truncation]] · [[interrupt-kills-background-processes]] · [[harness-credential-leaks-to-tools]]
