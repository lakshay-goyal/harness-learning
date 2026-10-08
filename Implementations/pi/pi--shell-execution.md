---
type: implementation
harness: pi
concept: shell-execution
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/bash.ts:22, packages/coding-agent/src/core/tools/bash.ts:40, packages/coding-agent/src/core/tools/bash.ts:97, packages/coding-agent/src/core/tools/bash.ts:186, packages/coding-agent/src/core/tools/bash.ts:255, packages/coding-agent/src/core/tools/powershell.ts:16, packages/coding-agent/src/utils/shell.ts:67, packages/coding-agent/src/utils/child-process.ts:49, packages/coding-agent/src/core/tools/output-accumulator.ts:40, packages/coding-agent/src/utils/output-files.ts:14, packages/coding-agent/src/core/bash-executor.ts:48, packages/durable/src/tools/bash.ts:8]
---
[[shell-execution]] in [[pi]].

`bash` (default-active) and `powershell` (opt-in, Windows-only) share one implementation `createShellToolDefinition` (`packages/coding-agent/src/core/tools/bash.ts`, `powershell.ts:49-57`). Foreground only, no default timeout, tail-truncated, full output spilled to a private temp file. Abbrev `packages/coding-agent/src/` = `packages/coding-agent/src/`.

## Mechanism

### Schema / text (model-facing)
- **Schema** (`packages/coding-agent/src/core/tools/bash.ts:40-43`): `command: string` — "Shell command to execute"; `timeout?: number` — "Timeout in seconds (optional, no default timeout)". No `description`, `cwd`, `run_in_background`, stdin param.
- **Description** (`bash.ts:255`, `${config.shellName}` = bash|PowerShell): "Execute a bash command in the current working directory. Returns stdout and stderr. Output is truncated to last 2000 lines or 50KB (whichever is hit first). If truncated, full output is saved to a temp file. Optionally provide a timeout in seconds." Limits interpolated from `DEFAULT_MAX_LINES/BYTES` ([[tool-description-design]]).
- **Snippet** "Execute bash commands (ls, grep, find, etc.)"; **guideline** "You can inspect PI_* environment variables for current model and session details." (`bash.ts:45-48`) — emitted only when `exposeSessionEnvironment` (default true, `bash.ts:250,257`). PowerShell: snippet "Execute PowerShell commands", identical guideline (`powershell.ts:18-21`); rules deduped by text ([[dynamic-tool-guidelines]]).
- Cross-tool rule in system prompt: if bash/powershell present and none of grep/find/ls → "Use bash for file operations like ls, rg, find" (`packages/coding-agent/src/core/system-prompt.ts:108-116`).
- `constrainedSampling: {type:"json_schema", strict:"prefer"}` (`bash.ts:260`; [[constrained-tool-sampling]]).
- **outputSchema** for programmatic callers (codemode) (`bash.ts:56-62`): `{output, truncated, full_output_path?, exit_code, wall_time_seconds}` — "A non-zero exit code is an error result for the model, but scripts still resolve to this value" (`:52-55`). `output` ≤ `STRUCTURED_OUTPUT_MAX_BYTES = 1 MiB` keeping first/last 512 KiB around `[... N bytes omitted ...]` (`OutputAccumulator.readFullOutput`, `output-accumulator.ts:149-176`; `1ff5b6fdd`). See [[structured-tool-output]].

### Timeout
- `resolveTimeoutMs` (`bash.ts:27-38`): undefined → no timeout; non-finite or ≤0 → "Invalid timeout: must be a finite number of seconds" (`85b7c2474`); > `MAX_TIMEOUT_MS = 2_147_483_647` (setTimeout int32 max) → "Invalid timeout: maximum is 2147483.647 seconds" (`cbcf4e04c` #6181 — previously clamped to an immediate timeout).
- On expiry: `killProcessTree`, then `throw "<partial output>\n\nCommand timed out after N seconds"` (`bash.ts:131-136,377-380`). Deliberate absence: [[no-bash-default-timeout]].

### Shell selection (`packages/coding-agent/src/utils/shell.ts:67-120`)
1. `shellPath` setting — must exist else throw `Custom shell path not found`.
2. Windows: `%ProgramFiles%\Git\bin\bash.exe`, `%ProgramFiles(x86)%\Git\bin\bash.exe`, then `where bash.exe` (spawnSync `where`, 5 s timeout); else throw with install guidance (Git for Windows / PATH / `shellPath`).
3. Unix: `/bin/bash`, `which bash`, fallback `{shell:"sh", args:["-c"]}`.
- Invocation `bash -c <command>` (`shell.ts:20-22`). Legacy WSL `C:\Windows\{System32|Sysnative}\bash.exe` → `args:["-s"]`, `commandTransport:"stdin"`, command written to stdin so `$VARS` expand in the target bash, not the Windows layer (`shell.ts:15-22`; `1287b69fe` #5893).
- PowerShell (`shell.ts:122-136`): Windows only ("The powershell tool is only available on Windows."); `pwsh.exe` preferred over `powershell.exe`; args `-NoProfile -NonInteractive -ExecutionPolicy Bypass -Command`; every command prefixed `try { [Console]::OutputEncoding=[System.Text.Encoding]::UTF8 } catch {}\n` (`powershell.ts:16,35`). Options subset: no `commandPrefix`/`shellPath` (`powershell.ts:29-30`).
- `shellCommandPrefix` setting prepended + `\n` to every bash command (`bash.ts:268`), e.g. `shopt -s expand_aliases\nsource ~/.bash_aliases` — non-interactive bash has no aliases (`packages/coding-agent/docs/shell-aliases.md:38-77`).

### Spawn (`createLocalShellOperations`, `bash.ts:97-166`)
- `signal.aborted` → throw early; `fsAccess(cwd)` → "Working directory does not exist: {cwd}\nCannot execute bash commands." (`1432fd91d` #479 — ENOENT from missing cwd/shell previously crashed the whole session as uncaught).
- `spawn(shell, [...args, command], {cwd, detached: platform !== "win32", env, stdio: [ignore|pipe(stdin transport), pipe, pipe], windowsHide: true})` (`bash.ts:110-116`). **stdin ignored** — no interactive input.
- `detached` → own process group on Unix; Windows not detached (`24dec9fcd`, commit #4013: detached broke pwsh.exe console stdio; `taskkill /T` doesn't need a group).
- pid tracked in a module Set (`trackDetachedChildPid`, `bash.ts:123,160`; `shell.ts:167-182`) → `killTrackedDetachedChildren()` on SIGHUP/SIGTERM/exit in interactive/print/rpc modes (`9b7948c4c`; `packages/coding-agent/src/modes/print-mode.ts:58`, `packages/coding-agent/src/modes/rpc/rpc-mode.ts:374`).
- Abort listener → `killProcessTree(pid)`; Unix `process.kill(-pid, SIGKILL)` (group) w/ fallback to pid; Windows `%SystemRoot%\System32\taskkill.exe /F /T /PID` by absolute path "so cleanup does not depend on PATH", spawn errors consumed (`shell.ts:187-218`; `7af2d27dc` #6596). **No SIGTERM grace** for the bash tool. Origin `6e9fa8dde` (2025-11-11): old `exec()` killed only the parent shell, so `sleep 4 && echo hello` survived abort. Concept: [[process-tree-kill]].

### Exit wait (`waitForChildProcess`, `packages/coding-agent/src/utils/child-process.ts:49-137`)
- Resolve on `close`; or after `exit` once both stdout/stderr `end`; else idle timer `EXIT_STDIO_GRACE_MS = 100` (`:16`) **re-armed on every data chunk** so a descendant still writing isn't cut off (`3fa409562` #5303/#5753), while a quiet inherited handle (Windows daemonized descendant, `close` never fires) still releases (`25b185f39` #2389 "agent-browser … spin forever").
- `error` event → reject (spawn failures become tool errors, not crashes).

### Exit status → result
- Signal-killed shell: `exitCode ?? (signalCode ? 128 + signum : 1)` (`bash.ts:155-158`); custom ops returning `null` → throw "Command terminated without an exit code" (`bash.ts:386-388`). `a8b3dd199` #9577: signal-terminated commands were reported as **successful** with partial output.
- **Non-zero exit is not thrown**: `content = output + "\n\nCommand exited with code N"`, `structuredContent`, `isError: true` (`bash.ts:400-407`) — so codemode scripts still get the structured value ([[tool-error-as-result]]; `8562bcf66`).
- Abort → throw `"<partial>\n\nCommand aborted"` (`bash.ts:374-376`). Timeout → throw (above). Both prepend partial output; empty-output text "" in these paths vs `(no output)` on normal completion (`bash.ts:338,373`).
- `wall_time_seconds` rounded to 0.1 s (`bash.ts:389`).

### Streaming & bounded accumulation
- `onUpdate({content:[]})` immediately (`bash.ts:318-320`); later updates throttled to `BASH_UPDATE_THROTTLE_MS = 100` (`renderers/bash.ts:19`; `bash.ts:303-316`), each a tail snapshot with `persistIfTruncated:true` (forces spill file as soon as truncated, `:286`). Tool result streaming added `7ac832586` (2025-12-16).
- **`OutputAccumulator`** (`packages/coding-agent/src/core/tools/output-accumulator.ts`, `6b18cdbac` commit #4165, replacing per-chunk `Buffer.concat` + `truncateTail` of the whole rolling buffer):
  - streaming `TextDecoder` (`:40,70`) — multibyte chars split across chunks decode correctly.
  - decoded tail of `maxRollingBytes = max(2*maxBytes,1)` (`:60`), trimmed when > 2× (`:191-193`); tracks whether tail starts on a line boundary, drops partial first line from snapshots (`:225,230-237`).
  - global line/byte counts incremental (`:182-211`) → notice line numbers are absolute (old impl counted only rolling buffer).
  - spill when raw bytes > maxBytes, decoded bytes > maxBytes, or lines > maxLines (`:239-243`); buffered raw chunks flushed first (`:245-256`); **raw bytes** written to the file.
  - line counting ignores trailing `\n` (`truncate.ts:47-56`; `f95306781` #4818).
- Truncation direction: **tail** (errors/results at the end; `truncate.ts:162-166`); `truncateTail` keeps end of an over-long last line, UTF-8-boundary safe (`truncate.ts:205-212,247-262`). See [[tool-output-truncation]].
- **Notice formats** (`bash.ts:338-356`):
  - partial last line: `[Showing last {X} of line {N} (line is {Y}). Full output: {path}]`
  - by lines: `[Showing lines {a}-{b} of {total}. Full output: {path}]`
  - by bytes: `[Showing lines {a}-{b} of {total} (50.0KB limit). Full output: {path}]`
  - empty: `(no output)`.
- **Spill files** ([[tool-output-spill]]): `packages/coding-agent/src/utils/output-files.ts` — `${tmpdir()}/<prefix>-<16 hex>.<ext>` (`:18-20`), mode `0o600` "Output can carry private data" (`:14-15`), flag `wx` never follows a pre-planted link (`:25-26,33`); prefixes `pi-bash`/`pi-powershell` `.log`; centralized `d677d0ee7`. No cleanup found (unverified).
- TUI renderer strips the model-facing footer so the path isn't duplicated (`renderers/bash.ts:52-60`; `7dad27e5f` #4819); preview last `BASH_PREVIEW_LINES = 5` visual lines (`renderers/bash.ts:18`); renderers split from implementation (`eb3e9feed`).
- **No ANSI stripping / binary sanitizing on the model path**: accumulator stores raw decoded text; `stripAnsi` + `sanitizeBinaryOutput` only in rendering (`render-utils.ts:48`) and in user `!` commands (`bash-executor.ts:78`). Never present on bash tool per `git log -S stripAnsi` (observed absence).

### Environment ([[env-vars-as-context]])
- `getShellEnv()` = `process.env` + pi bin dir (managed rg/fd) prepended to PATH, case-insensitive PATH key (`shell.ts:138-150`).
- `resolveSpawnContext` (`bash.ts:186-212`): deletes inherited `PI_SESSION_ID/FILE/PROVIDER/MODEL/REASONING_LEVEL`, then sets from ctx when `exposeSessionEnvironment` (`bb3d7d399` #6967). Not injected into user `!`/`!!` (`docs/environment-variables.md:49`). `PI_CODING_AGENT=true` process-wide (`packages/coding-agent/src/cli/setup.ts:6`).

### Extension seams ([[pluggable-tool-backends]])
- `BashOperations.exec(command, cwd, {onData, signal, timeout, env}) → {exitCode}` (`bash.ts:75-94`; `9ed88646a` 2026-01-08 "pluggable operations for remote tool execution"); `createLocalBashOperations({shellPath})` exported (`9651e4114` #2299) for `user_bash` interceptors.
- `spawnHook(ctx)→ctx` rewrites command/cwd/env (`bash.ts:184,211`; `86b43c8ea`).
- cwd = `ctx?.cwd || cwd` (`bash.ts:271`; `62835ea81` #8627 — extension-registered tools followed creation-time cwd).

### User `!cmd` path (not a model tool)
- `packages/coding-agent/src/core/bash-executor.ts:48-152`: strips ANSI, `sanitizeBinaryOutput` (control chars + U+FFF9–FFFB which crash string-width, `shell.ts:152-161`), removes `\r`; rolling buffer 2×50KB; spill > 50KB; result converted to user message "Ran `cmd`" + fenced output (see [[message-conversion-layer]]); `!!cmd` excluded from context (`docs/usage.md:76`). UTF-8 corruption fixes `6ddfd1be1` (#433), `7293d7cb8` (#608, streaming decoder); binary crash `ad42ebf5f`; temp stream fd leak `e4f847ff6`. `user_bash` hook fails closed (`509ee2bd0`).

### `exec.ts` (extension helper) & output guard
- `packages/coding-agent/src/core/exec.ts:34-107`: `spawn(command, args, {shell:false})`, SIGTERM then SIGKILL after 5 s (`:52-63`), never rejects (`code: 1` on error, `:99-105`).
- `packages/coding-agent/src/core/output-guard.ts:45-70`: print/json/rpc modes redirect `process.stdout.write` to stderr so tool/extension stray writes can't corrupt protocol (`f1fe49a64` #2482); raw writes retry `ENOBUFS/EAGAIN/EWOULDBLOCK` every 10 ms (`:9,20-43`; `ce0e801d8` after revert `9600ded92`).

### Absences
- No background mode / job param / `nohup` handling; detached descendants killed on pi shutdown — [[no-background-bash]] ("Use tmux", `271810c80`).
- No sandbox, no approval prompt: "does not ask for approval before every tool call" (`docs/security.md:3`); gating only via `tool_call` hooks ([[tool-call-gate]], [[no-sandbox]], [[no-permission-prompts]]).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_TIMEOUT_MS` | 2_147_483_647 ms | `packages/coding-agent/src/core/tools/bash.ts:22` |
| default timeout | none | `bash.ts:42` |
| `STRUCTURED_OUTPUT_MAX_BYTES` | 1 MiB (512 KiB head + tail) | `bash.ts:24` |
| `DEFAULT_MAX_LINES` / `DEFAULT_MAX_BYTES` (tail) | 2000 / 50 KiB | `packages/coding-agent/src/core/tools/truncate.ts:11-12` |
| `EXIT_STDIO_GRACE_MS` | 100 ms (re-armed per chunk) | `packages/coding-agent/src/utils/child-process.ts:16` |
| `BASH_UPDATE_THROTTLE_MS` | 100 ms | `packages/coding-agent/src/core/tools/renderers/bash.ts:19` |
| `BASH_PREVIEW_LINES` | 5 | `renderers/bash.ts:18` |
| accumulator rolling tail | max(2×maxBytes,1) | `output-accumulator.ts:60` |
| spill file mode | 0o600, flag `wx` | `packages/coding-agent/src/utils/output-files.ts:14-15,25-26` |
| `where` lookup timeout | 5000 ms | `packages/coding-agent/src/utils/shell.ts` (findExecutableOnPath) |
| `exec.ts` kill grace | SIGTERM → SIGKILL after 5 s | `packages/coding-agent/src/core/exec.ts:52-63` |
| output-guard retry | 10 ms | `packages/coding-agent/src/core/output-guard.ts:9` |
| `!` command preview lines | 20 | `packages/coding-agent/src/modes/interactive/components/bash-execution.ts:19` |
| durable `MAX_TIMEOUT_SECONDS` | 2_147_483_647/1000 | `packages/durable/src/tools/bash.ts:8` |

## Evolution
- 2025-10-17 `ffc9be886`: v0 desc "…Commands run with a 30 second timeout."
- 2025-11-11 `6e9fa8dde`: spawn detached + kill process group (abort left `sleep 4 && echo hello` running).
- 2025-11-12 `29900ce64`: timeout optional, no default ("commands run until completion unless specified"); desc "Optionally provide a timeout in seconds." (30 s killed long builds — inferred).
- 2025-12-07 `de77cd141` / `306f9cc66` / `b813a8b92` (#134): tail truncation 2000 lines / 30KB→50KB, full output to temp file, notices model-visible.
- 2025-12-08 `ad42ebf5f`: binary output crash in bash mode. 2025-12-16 `7ac832586`: tool result streaming.
- 2026-01-04/10 `6ddfd1be1` (#433) / `7293d7cb8` (#608): streaming UTF-8 decoding (user `!` / remote).
- 2026-01-05 `1432fd91d` (#479, #1230): cwd check + `error` handler.
- 2026-01-08 `9ed88646a`: `BashOperations`. 2026-02-01 `86b43c8ea`: `spawnHook`. 2026-03-18 `9651e4114`: `createLocalBashOperations`.
- 2026-03-19 `25b185f39` (#2389): don't wait on inherited handles.
- 2026-03-22 `f1fe49a64` (#2482): stdout takeover.
- 2026-04-05 `52d16d5a3` (#2852): spill on any truncation (was only when bytes > 50KB → line-limit notice without file).
- 2026-04-16 `9b7948c4c`: kill tracked detached children on shutdown. 2026-04-27 `e4f847ff6`: close temp streams.
- 2026-05-01 `24dec9fcd` (#4013): Windows not detached.
- 2026-05-04 `6b18cdbac` (#4165): `OutputAccumulator`. 2026-05-21 `f95306781` (#4818) trailing-newline count; `7dad27e5f` (#4819) renderer footer dedupe.
- 2026-05-24 `9600ded92` → `ce0e801d8`: RPC stdout backpressure retry (reverted then re-landed).
- 2026-06-15 `3fa409562` (#5753): idle grace re-armed per chunk.
- 2026-06-19 `1287b69fe` (#5893): legacy WSL via stdin.
- 2026-06-30 `cbcf4e04c` (#6181) / 2026-07-01 `85b7c2474`: reject oversized / non-positive timeouts.
- 2026-07-22 `bb3d7d399` (#6967): PI_* env + imperative guideline → 2026-08-06 `4e64de695` (#7128) softened "You can inspect…" ("reduce unnecessary inspection commands", CHANGELOG `:759`; exact commands unverified; [[guideline-softening]]).
- 2026-08-24 `80e62761f` (#8512): PowerShell tool; `${config.shellName}` in description.
- 2026-08-26 `7af2d27dc` (#6596): taskkill absolute path; `eb3e9feed`: renderers split.
- 2026-09-01 `62835ea81` (#8627): `ctx.cwd`.
- 2026-09-05 `fcff255b0`: strict-prefer constrained sampling default.
- 2026-09-17 `a8b3dd199` (#9577): 128+signal; null exit → error.
- 2026-09-29 `8562bcf66`: non-zero exit → `isError` return + `structuredContent` (codemode); `1ff5b6fdd`: 1 MiB structured output; output-schema description "Combined stdout and stderr, up to 1 MiB…" later removed in `6f1072cc0` (2026-10-01, codemode prompt shrink).
- 2026-10-04 `d677d0ee7`: output files centralized.
- 2026-09-24→10-05 durable: `601437d5a` truncation defaults, `445770e03` (Package 16) bash tool, `cdf79797b` windowed output + counted skips, `cd60a5b99` output BOM, `68c22123b` powershell over argv exec, `1543dd8f6` Windows taskkill retry.

## Evidence commits
`ffc9be886` `6e9fa8dde` `29900ce64` `de77cd141` `306f9cc66` `b813a8b92` `ad42ebf5f` `7ac832586` `6ddfd1be1` `7293d7cb8` `1432fd91d` `9ed88646a` `86b43c8ea` `9651e4114` `25b185f39` `f1fe49a64` `52d16d5a3` `9b7948c4c` `e4f847ff6` `24dec9fcd` `6b18cdbac` `f95306781` `7dad27e5f` `9600ded92` `ce0e801d8` `3fa409562` `1287b69fe` `cbcf4e04c` `85b7c2474` `bb3d7d399` `4e64de695` `80e62761f` `7af2d27dc` `eb3e9feed` `62835ea81` `fcff255b0` `a8b3dd199` `8562bcf66` `1ff5b6fdd` `6f1072cc0` `d677d0ee7` `601437d5a` `445770e03` `cdf79797b` `cd60a5b99` `68c22123b` `1543dd8f6`

## Quirks
- Raw ANSI escapes reach the model (no strip on tool path) — intentional? no commit discusses it (open question).
- Timeout/abort errors lose `structuredContent` (thrown, not returned) while non-zero exit keeps it.
- Spill temp files never deleted by pi (no cleanup in `output-files.ts`; unverified elsewhere).
- Fixed post-exit grace is idle-based: a descendant that keeps writing at <100 ms intervals keeps the tool call alive indefinitely (inferred from `child-process.ts` design).
- Same `PI_*` guideline text for bash & powershell relies on rule dedupe (`system-prompt.ts:93-100`).
- `PI_EXPERIMENTAL` gate for strict schemas removed by `fcff255b0`; whether it gates anything else tool-related is unverified.

## Durable variant (packages/durable)
- `packages/durable/src/tools/bash.ts`: schema `command` ("Bash command to execute" / "PowerShell command to execute"), `timeout?` same text (`:10-18`); description "…Returns **combined** stdout and stderr…" (`:128`, powershell `:152`).
- Timeout validated finite >0 ≤ 2^31-1 ms (`:8,46-54`).
- Runs through `env.exec` ([[pluggable-tool-backends]]): `onOutput → api.output(text, info.skipped)`, `spill {afterBytes: 50KB, afterLines: 2000}`, `window: api.outputWindow` (`:86-100`). Harness computes `outputWindow` for `retain:"tail"` tools: `{maxBytes, maxLines, minIntervalMs: 100, bytesPerSecond: 100 KiB}` (`packages/durable/src/harness/tool.ts:198-206`, `harness/output.ts:261`); remote env drops out-of-window output at the source with exact `skipped{bytes,newlines}` counts (`cdf79797b`); `NodeExecutionEnv` ignores `window`.
- `outputLimits:{retain:"tail"}` (`:130`); spill path → `info` diagnostic `full_output` "Full output: {path}" (`:105-108`), not content ([[harness-diagnostics-channel]]).
- **Non-zero exit / timeout are thrown** (`Command exited with code N`, `Command timed out after N seconds`, `Command aborted`) → error result still carrying output (`:109-116`) — opposite of coding-agent's `isError` return.
- Hooks: `commandPrefix`, `prepare(execution)` may rewrite command/cwd/env/inheritEnv (`:23-71`).
- `powershell`: tries `["pwsh","powershell"]` as **argv** (no outer shell), next program only on `spawn_error`; script = UTF-8 prefix + command; same `POWERSHELL_ARGS` (`:140-164`; `68c22123b`).
- `NodeExecutionEnv`: POSIX `detached` + `killProcessTree` SIGKILL `-pid` / taskkill; `activeChildPids` for `cleanup()`; throwing `onOutput` → `callback_error` kills command; spill backpressure pauses child; spill failure kills with "Failed to preserve complete shell output" (`packages/durable/src/env/node.ts:274-298,610,711-797`); `StreamDecoder` drops U+FEFF only at stream start (`env/decode.ts:1-29`; `a19c09d9b`).

## Failures
- [[bash-spawn-errors-crash-session]]
- [[signal-killed-command-reported-success]]
- [[bash-descendants-hang-or-lose-output]]
- [[bash-output-integrity]]
- [[bash-timeout-clamped-to-immediate]]
- [[windows-process-tree-and-shells]]
- [[tools-ignore-session-cwd]]
