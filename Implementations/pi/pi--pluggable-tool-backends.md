---
type: implementation
harness: pi
concept: pluggable-tool-backends
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/bash.ts:75-97, packages/coding-agent/src/core/tools/bash.ts:174-272, packages/coding-agent/src/core/tools/read.ts:48-55, packages/coding-agent/src/core/tools/edit.ts:83-100, packages/coding-agent/src/core/tools/write.ts:27-35, packages/coding-agent/src/core/tools/grep.ts:50-74, packages/coding-agent/src/core/tools/find.ts:49-128, packages/coding-agent/src/core/tools/ls.ts:34-41, packages/coding-agent/src/core/tools/index.ts:1-60, packages/coding-agent/examples/extensions/ssh.ts, packages/coding-agent/examples/extensions/gondolin/index.ts:1-122, packages/durable/src/env/index.ts:1-324, packages/durable/src/env/node.ts:50-1224, packages/durable/src/tools/env.ts:4-8, packages/env/README.md:7-41]
---
[[pluggable-tool-backends]] in [[pi]].

## Mechanism
### Coding-agent: per-tool `*Operations`
- Every built-in tool factory takes `options.operations`; default = local Node fs / spawn (`packages/coding-agent/src/core/tools/index.ts:1-60` exports `BashOperations, EditOperations, FindOperations, GrepOperations, LsOperations, PowerShellOperations, ReadOperations, WriteOperations` + `createLocalBashOperations`, `createLocalPowerShellOperations`).
  - `BashOperations.exec(command, cwd, {onData(Buffer), signal?, timeout?, env?}) → {exitCode: number|null}` — "Report signal terminations as 128 + signal number; a null exit code is treated as a failed command" (`bash.ts:75-94`). Local impl `createLocalShellOperations(shellName, resolveShellConfig)` (`bash.ts:97-172`), `createLocalBashOperations({shellPath})` (`bash.ts:174-175`; exported `9651e4114` #2299).
  - `ReadOperations {readFile→Buffer, access, detectImageMimeType?}` (`read.ts:48-55`); `EditOperations {readFile, writeFile, access}` (`edit.ts:83-90`); `WriteOperations {writeFile, mkdir}` (`write.ts:27-32`); `LsOperations {exists, stat, readdir}` (`ls.ts:34-41`).
  - `GrepOperations {isDirectory, readFile}` only (`grep.ts:53-58`) — **ripgrep always spawns locally** (`grep.ts:119,168`), ops cover only dir check + context-line reads.
  - `FindOperations {exists, glob?(pattern, cwd, {ignore, limit})}` (`find.ts:52-57`); if custom `glob` provided it replaces fd, with ignore `**/node_modules/**`, `**/.git/**` (`find.ts:113-128`).
- **`spawnHook(ctx) → ctx`** rewrites command/cwd/env before exec (`bash.ts:184-211,224`; `86b43c8ea` 2026-02-01); `commandPrefix`, `shellPath` options (`bash.ts:268`). Example `examples/extensions/bash-spawn-hook.ts`.
- Effective cwd = `ctx?.cwd || cwd` for all 8 tools (`62835ea81` #8627) so swapped-in backends follow session cwd ([[path-normalization]]).
- **User shell path**: extension `user_bash` event may return `{operations}` (custom executor) or `{result}` (full replacement); error/invalid result **aborts** — fail-closed so a broken remote-exec extension never silently runs locally (`packages/coding-agent/src/core/extensions/types.ts:1123-1132,1436-1449`; `runner.ts:1262-1296`; `509ee2bd0` #9662).
- **Examples** → [[tool-only-isolation]]:
  - `examples/extensions/ssh.ts`: "When --ssh is provided, read/write/edit/bash run on the remote" (`ssh.ts:1-14`).
  - `examples/extensions/gondolin/`: runs built-in tools + `!` commands in a local Gondolin micro-VM, host cwd mounted at `/workspace` (`gondolin/index.ts:1-7,84-122`; `86314bf38`); caveat "Commands inside the VM inherit the host process environment … Do not use this pattern as a credential boundary" (`docs/containerization.md:157`).
  - `examples/extensions/sandbox/`: OS sandbox (sandbox-exec / bubblewrap) by **overriding** the built-in `bash` tool (`sandbox/index.ts:1-11`).
- Built-in tools are "just" extension-style definitions (`235b247f1`: "Built-in tools now work like custom tools"), so a same-named extension tool replaces them ([[plugin-tools]]).

### Durable: `ExecutionEnv` (packages/durable/src/env)
- `ExecutionEnv extends FileSystem, Shell` (`packages/durable/src/env/index.ts:324`). **Result style**: expected failures returned as `Result<T,E>`, never thrown (`index.ts:3-21`).
  - `FileErrorCode` (8): aborted, not_found, permission_denied, not_directory, is_directory, invalid, not_supported, unknown (`index.ts:35-43`). `ExecutionErrorCode` adds callback_error, shell_unavailable, spawn_error, timeout; may carry `spillPath` (`index.ts:57-68`).
- **FileSystem** (`index.ts:171-236`): `id` = file namespace — "Every local Node environment shares one id; each container or remote host has its own" (`packages/durable/src/env/index.ts:172-176`); local id `node:local` (`node.ts:604-605`). Ops: `absolutePath, joinPath, readTextFile, openTextLineReader, readTextLines{maxLines}, readBinaryFile, openBinaryReader{noFollow}, writeFile, appendFile, truncateFile, flushFile, renameFile, fileInfo, listDir, openDirReader, watch, canonicalPath, exists, createDir, remove{recursive,force}, createTempDir, createTempFile, cleanup`.
  - `BinaryReader`: positional reads on one *opened* file, valid across renames; `info()` = opened file (`index.ts:95-108`); `scanLines({startLine,endLine}) → LineScan{newlines,start,end,firstLineEnd,lastLineStart,selectedBytes,firstLineBytes}` single pass, decoded sizes equal whole-file `TextDecoder` (BOM not counted) (`index.ts:101-106,140-157`) — powers bounded [[file-read-tool]].
  - `DirReader.next(maxEntries)` paged, vanished entries skipped (`index.ts:159-168`).
  - `watch`: missing targets allowed; recursive doesn't follow symlinks; changes `{paths}` (may be spurious, never missed while healthy) / `{overflow:true}` rescan / `{error}` terminal; coverage established before `watch()` returns (`index.ts:110-138,208-217`).
- **Shell.exec(command: string | argv[], options, ctx)** (`index.ts:238-322`): string → env's shell; array → `command[0]` directly, no shell. Abort/timeout kills only this command's processes; `cleanup()` kills all (owner shutdown). Options `cwd, env, inheritEnv, timeout (s), onOutput(text, ctx, {stream, skipped?}), spill{afterBytes,afterLines}, window`. `onOutput` raw, unbounded per chunk, per-stream decoding. **`window`** lets the env drop output outside the caller's retained tail, reported as `skipped{bytes,newlines,endsWithNewline}` (U+FFFD counted as 3 bytes) — guaranteed never in the kept tail (`index.ts:262-306`; `cdf79797b`). `NodeExecutionEnv` ignores `window`; only the remote daemon uses it.
- **NodeExecutionEnv** (`node.ts`, 1226 lines): POSIX children `detached`, `killProcessTree` SIGKILL `-pid` / Windows `taskkill` (`node.ts:274-298,805`) — [[process-tree-kill]]; `activeChildPids` for `cleanup()` (`node.ts:610,1221-1224`); throwing `onOutput` → `callback_error` + kill (`node.ts:711-728`); spill pauses output while the temp file is created, honours backpressure, spill failure kills command "Failed to preserve complete shell output" (`node.ts:741-797`); temp files `<tmp>/tmp-XXXX/<prefix><uuid><suffix>` (`node.ts:1210-1219`).
- **Durable tools** get env from `api.env`; none → ordinary error "No execution environment is configured" (`packages/durable/src/tools/env.ts:4-8`). They never touch `node:fs`, so they run unchanged on `RemoteExecutionEnv` (pi-env Rust daemon over SSH) — [[remote-execution-env]], [[remote-host-trust]].
- **Parity contract**: pi-env "results must match Durable's `NodeExecutionEnv` … Only error messages may differ" (`packages/env/README.md:7-9`); enforced by `createEnvConformance` (`packages/durable/src/testing/env-conformance.ts`) + `packages/env/test/differential.test.ts` running random op sequences **and Durable's tools** against both envs (`packages/env/README.md:39-41`). Read differential: `test/tools-read-differential.test.ts` keeps pre-bounded `referenceRead` as oracle; 400 seeded random files × 4 (offset,limit) trials incl. BOM, CRLF, invalid UTF-8, split multibyte, lines > 50 KB, NaN/-4/2.5/1e20 (`:13-16,100-130,155-178`).
- Mutation queue key `${env.id}\0${canonicalPath}` → local and remote files with the same path never block each other ([[per-file-mutation-queue]]).

## Constants
| name | value | path:line |
|---|---|---|
| local env id | `node:local` | `packages/durable/src/env/node.ts:605` |
| `MAX_TIMEOUT_MS` (Node env) | 2^31−1 | `packages/durable/src/env/node.ts:50` |
| `EXIT_STDIO_GRACE_MS` (Node env) | 100 | `node.ts:52` |
| `SPILL_HIGH_WATER_MARK` / `BINARY_READ_CHUNK` | 1 MiB / 1 MiB | `node.ts:53-55` |
| durable `READ_CHUNK` | 64 KiB | `packages/durable/src/tools/read.ts:39` |
| read differential trials | 400 files × 4 | `packages/durable/test/tools-read-differential.test.ts:155-178` |

## Evolution
- 2026-01-08 `9ed88646a` — "add pluggable operations for remote tool execution" (`BashOperations` etc.).
- 2026-02-01 `86b43c8ea` — bash `spawnHook`. 2026-03-18 `9651e4114` (#2299) export local bash operations. 2026-03-22 `235b247f1` built-ins as ordinary definitions.
- 2026-05-14 `846906e4d` — "result-based execution env" in the old agent harness (`packages/agent/src/harness/env/nodejs.ts`). 2026-06-03 `86314bf38` containerization guide + Gondolin. 2026-09-01 `62835ea81` (#8627) ctx.cwd. 2026-09-16 `509ee2bd0` (#9662) `user_bash` fail-closed.
- 2026-09-24 `601437d5a`, 2026-09-29 `445770e03` (Package 16) — durable env + read/bash/edit/write tools. 2026-10-01 `7fd478a2e` old harness tools moved into `packages/durable/src/tools`.
- 2026-10-04 `4748c627a` bounded binary/dir readers + argv exec; `cdf79797b` windowed output with counted skips; `a19c09d9b` bounded `read` via `scanLines`. 2026-10-05 `ba03e03f2` pi-env `RemoteExecutionEnv` over SSH; `cd60a5b99` growing reads, output BOM.

## Evidence commits
`9ed88646a`, `86b43c8ea`, `9651e4114`, `235b247f1`, `846906e4d`, `86314bf38`, `62835ea81`, `509ee2bd0`, `601437d5a`, `445770e03`, `7fd478a2e`, `4748c627a`, `cdf79797b`, `a19c09d9b`, `ba03e03f2`, `cd60a5b99`.

## Quirks
- grep is not remotable via ops (local `rg` spawn), unlike find (custom `glob`) — an SSH/VM backend silently searches the host fs for `grep` `(inferred from grep.ts:119-168)`.
- Coding-agent mutation queue keys on local `realpath` with no namespace id, so remote backends share keys with same-named local paths (durable fixes this).
- No production code constructs `RemoteExecutionEnv` at HEAD (10-durable open question) `(unverified intent)`.

## Failures
- [[tools-ignore-session-cwd]]
- [[read-fails-on-growing-file]]
- [[bash-output-integrity]]
