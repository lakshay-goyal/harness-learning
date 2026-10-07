---
type: implementation
harness: pi
concept: remote-execution-env
commit: b30a6dd77
files: [packages/env/README.md:3-41, packages/env/docs/protocol.md:3-114, packages/env/docs/semantics.md:3-65, packages/env/daemon/src/main.rs:33-516, packages/env/daemon/src/frame.rs:1-99, packages/env/daemon/src/output.rs:1-99, packages/env/daemon/src/exec.rs:21-481, packages/env/daemon/src/watch.rs:21-49, packages/env/src/connection.ts:5-363, packages/env/src/remote-env.ts:33-863, packages/env/src/ssh.ts:59-530, packages/env/src/watch.ts:16-157, packages/durable/src/env/index.ts:110-322, packages/durable/src/env/node-watch.ts:10-474]
---
[[remote-execution-env]] in [[pi]].

## Mechanism
- `@earendil-works/pi-env` v1.0.4: "Remote execution environments for Pi Durable: a small daemon deployed over SSH and its ExecutionEnv client" (`packages/env/package.json:3-4`). Tools run remotely; Durable worker, storage, credentials stay local (`packages/env/README.md:3-4`).
- **Contract**: results must equal pi-durable's `NodeExecutionEnv` run on the remote machine; only error messages may differ (`packages/env/README.md:7-9`; `docs/semantics.md:3-4`). Enforced by Durable's env conformance suite + `test/differential.test.ts` (random op sequences + Durable's tools against Remote and Node envs) (`packages/env/README.md:40-43`) → [[pi--harness-evals]]. Durable tools never touch `node:fs` (`packages/durable/src/tools/env.ts:4-8`), so they run unchanged → [[pluggable-tool-backends]].
- **Consumers**: none in the monorepo at HEAD (grep for `pi-env`, `RemoteExecutionEnv`, `connectSsh` outside `packages/env` hits only lockfile/tsconfig); plugged via `HarnessOptions.env` (`packages/durable/src/harness/types.ts:176`). Remote path `~/.pi/mobile/tools/` and "a phone that lost its network" comment suggest a mobile host (unverified).
- **Daemon** (Rust 2024, toolchain 1.96.0; deps only `encoding_rs`, `notify`, `serde_json` + `libc`/`windows-sys`, all `=`-pinned; LTO, codegen-units=1, strip; `PI_ENV_VERSION` baked by `build.rs`) (`daemon/Cargo.toml:8-29`; `rust-toolchain.toml:2`; `build.rs:1-12`). Prebuilt per platform (Linux, macOS, Android/Termux, Windows × x86-64/arm64) in npm `bin/` (`packages/env/README.md:30,36`; `packages/env/package.json:17-21`).
- **Wire protocol v1** (`docs/protocol.md`):
  - `pi-env serve --token <hex>` over stdin/stdout; stderr diagnostics only (`:3-4`; `main.rs:491-516`).
  - Sync line `PI-ENV <token>\n` first; client discards pre-sync shell-rc noise (≤1 MiB kept) (`main.rs:438-443`; `connection.ts:327-342`). Token = 16 random bytes hex — a sync marker, NOT auth (daemon never checks it) (`connection.ts:267`).
  - Frame `u32 length | u8 type | u32 id | u32 jsonLength | json | payload`, big-endian both ways; raw bytes in payload (no base64) (`protocol.md:14-25`; `frame.rs:84-99`; `connection.ts:97-105`). `MAX_FRAME` 16 MiB; response payload ≤ `MAX_FRAME − 64 KiB`; `pread` clamped (`frame.rs:14-16`; `main.rs:210`). Bad JSON → `EINVAL` reply, framing intact; length outside `9..=MAX_FRAME` ends connection (`frame.rs:58-74`); client tears session down on corrupt frame (`connection.ts:345-363`).
  - Types 1 request, 2 result, 3 error, 4 event, 5 cancel, 6 ping (`frame.rs:6-11`).
  - Errors `{code, message, syscall?, path?}`; code = Node errno or `shell_unavailable|spawn_error|timeout|aborted|unknown`; only codes are contract (`protocol.md:37-39`); client maps to Durable `FileError` like `NodeExecutionEnv` (`ENOENT→not_found`, `EACCES/EPERM→permission_denied`, `ENOTDIR`, `EISDIR`, `EINVAL/SYMLINK/NOT_REGULAR→invalid`, else `unknown`) (`packages/env/src/errors.ts:9-34`).
  - Paths absolute on the wire; client resolves `~`, `file://`, relative with REMOTE OS rules incl. Windows drive-relative via `driveCwds` (`remote-env.ts:371-399`); lone surrogates → U+FFFD (`connection.ts:92-95`).
  - Ops: `hello {protocol}` first → `{protocol, version, os, arch, home, separator, tmpdir, cwd, driveCwds, pid}`, client rejects protocol ≠ 1 (`main.rs:143-154`; `connection.ts:296-297`); FS `lstat, realpath, write{append,parents,keep}, writeChunk, truncate, fsync, rename, mkdir, rm, mkdtemp, open{noFollow,mode}, pread, fstat, scanLines, opendir, readdir{max}, close`; `exec {command|argv, cwd, env, inheritEnv, shellPath?, timeoutMs?, spill?, window?}` streams `event{kind:"output", stream, skipped?}` → `{exitCode, spillPath?}`; `watch {targets, mode?, pollIntervalMs?, maxDirectories?}` → `ready{mode}`, `change{paths}`, `change{overflow:true, mode:"polling"}`, terminal `error` (`protocol.md:67-114`).
  - Handles numeric, per connection, ≤4096 (`EMFILE`) (`main.rs:38-39,111-119`); failed `writeChunk` poisons handle (`EBADF`, no gaps) (`main.rs:46-47,166-176`); per-handle FIFO chunk queue (`main.rs:362-387,412-423`).
  - Cancellation: `Control` registered before request runs (no lost cancel) (`main.rs:394-397`; `exec.rs:39-58`); default abort, `{mode:"kill"}` kills without abort (`protocol.md:45-50`); `scanLines` checks abort per 64 KiB (`main.rs:284-305`).
- **Scheduling / flow control**: 16-worker pool for file ops; `exec`/`watch` own threads with 16 MiB stacks (`main.rs:36-37,329-410`). Single stdout writer with **control queue (results, errors, pings, watch events) always drained before bulk**; replies with payload > 64 KiB (`CONTROL_PAYLOAD`) and command output go to bulk (`output.rs:1-4,71-99`; `main.rs:40-41,347-360`). Unwindowed output blocks while > 4 MiB (`BULK_LIMIT`) bulk unsent — like a full pipe (`output.rs:13-14,51-61`; `exec.rs:225`).
- **Output window at source**: `ShellOutputWindow{maxBytes,maxLines,minIntervalMs,bytesPerSecond}` ported from Durable (`daemon/src/window.rs:1-28`); ≤1 unsent output frame per command (20 ms poll, `exec.rs:22-23`); text that can never land in the caller's kept tail is dropped and reported as `skipped{bytes,newlines,endsWithNewline}` (`protocol.md:104-107`) → never crosses SSH. `NodeExecutionEnv` ignores `window` (grep) → [[shell-execution]].
- **Liveness**: both sides ping every 5 s, tear down after 30 s silence (`main.rs:33-34`; `connection.ts:13-15,288-294`); daemon counts ANY received bytes via `Seen<R>` "so a large frame arriving slowly counts as a live client" (`main.rs:313-327`); on silence/EOF daemon kills every process group and exits — "A client that went silent (a phone that lost its network) leaves nothing running" (`main.rs:464-488`) → [[process-tree-kill]]. Client allows 60 s for start + hello (`connection.ts:16-17,271-281`).
- **Sessions / reconnect**: each daemon start = numbered `Session`; in-flight requests on a lost session fail `{code:"unknown", lost:true}` (`isConnectionLost`) (`connection.ts:107-115,307-320`); next request starts a new daemon lazily, failed start not remembered (`:214-225`); handle-bound requests carry `session` and fail after reconnect (`:85-90,190-200`); mutation outcome unknown on loss (`:164-167`).
- **Exec**: shell `shellPath` (missing → `shell_unavailable`) → `/bin/bash` → `which bash` → `sh -c`; Windows Git Bash → `where bash.exe`, legacy WSL special-cased (`sys/unix.rs:228-247`; `sys/windows.rs:333-412`); argv runs directly, Windows batch files refused (`semantics.md:43-46`). Unix `setsid()` + default signals + empty mask, kill `SIGKILL -pgid` (`sys/unix.rs:266-295`); Windows Job Object `KILL_ON_JOB_CLOSE` (`sys/windows.rs:524-548`); signalled exit = 128+signal (cleanup → 137). Drain until both pipes EOF or 100 ms idle (`EXIT_STDIO_GRACE`) (`exec.rs:21,417-423`). Spill full raw output to `<tmpdir>/tmp-XXXX/pi-output-<uuid>.log` past `spill.afterBytes|afterLines`, path also on timeout/abort errors (`exec.rs:451-481`) → [[tool-output-spill]]. Env: `inheritEnv` = daemon env → `shellEnv` → `env`; else only `env` (`semantics.md:48-49`; `remote-env.ts:858-863`). Decoding WHATWG UTF-8 per stream via `encoding_rs`, only leading BOM dropped, no UTF-16 sniff (`decode.rs:51-77`).
- **Watch coverage contract** (`packages/durable/src/env/index.ts:110-138,208-217`): missing targets allowed (creation = change); `recursive` doesn't follow symlinks; `exclude{hidden,names}`; `WatchChange` = `{paths}` (maybe spurious, never missed while healthy) | `{overflow:true}` (rescan) | `{error}` (terminal); `native` ≈2 s latency, `polling` can miss change-then-undo; coverage established by the time `watch()` returns. `NodeFileWatcher`: native events only trigger debounced rescans; snapshot diff + event paths (`node-watch.ts:258-266`); polling default on Windows ("native watchers keep directories open and so block renaming their parents") and network/FUSE fs (`:10-13`); per-directory watchers on Linux/Android, recursive root elsewhere (`:415-421`); out of watches → polling + `overflow` (`:469-474`). Daemon `watch.rs` is a port running next to the files with identical constants (`watch.rs:21-49`). Client `RemoteWatcher` reopens with backoff 1 s → 30 s then emits `{overflow:true}`; throwing callback doesn't stop watching (`src/watch.ts:16-17,70-157`).
- **SSH bootstrap** (`src/ssh.ts`; details → [[remote-host-trust]], [[supply-chain-pinning]]): `-T -a -x`, `BatchMode=yes`, `ClearAllForwardings`, `ForwardAgent=no`, `ForwardX11=no`, `ControlMaster=no`, `ControlPath=none`, `RemoteCommand=none`, `PermitLocalCommand=no`, `SendEnv=-*`, `ServerAliveInterval=15`, `StrictHostKeyChecking=yes` with app-owned `UserKnownHostsFile`, `GlobalKnownHostsFile=none`, `HostKeyAlias`, `IdentitiesOnly=yes`, `--` (`:77-120`); arg-injection guards (`:59-70`); explicit TOFU `scanHostKey` → `acceptHostKey`, changed key refused (`HostKeyChangedError`) (`:177-286`); POSIX probe with `PI-ENV-PROBE` marker, PowerShell `-EncodedCommand` fallback on any non-host-key failure, Termux warnings (`:288-353`); content-addressed `~/.pi/mobile/tools/pi-env-<sha256[0:32]>`, refuses deploy without a hash tool, temp + verify + `chmod 700` + `mv -f`, prune old; Windows base64 lines + `PI-ENV-END` via PowerShell `$input`, 20 × 250 ms rename retries; verify before every start (`:369-502`); login shell `exec "$SHELL" -lc` option (`:465-475`); lazy `sshConnection()` (failures → op `spawn_error`, next op retries) vs eager `connectSsh()` (`:504-530`).
- **Sandboxing: none** — daemon has full SSH-user rights; no allowlist/jail (grep); isolation comes from where it runs + SSH; process-group kill on disconnect is the only containment → [[no-sandbox]], [[tool-only-isolation]]. Commands don't inherit local env (unlike Gondolin VM extension, `docs/containerization.md:153-183`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_FRAME` | 16 MiB | `daemon/src/frame.rs:14`, `src/connection.ts:12` |
| ping / silence / start timeout | 5 s / 30 s / 60 s | `src/connection.ts:12-17`, `daemon/src/main.rs:33-34` |
| worker pool / exec stack | 16 threads / 16 MiB | `main.rs:36-37` |
| `MAX_HANDLES` | 4096 | `main.rs:38-39` |
| `CONTROL_PAYLOAD` | 64 KiB | `main.rs:40-41` |
| `BULK_LIMIT` | 4 MiB | `output.rs:13-14` |
| output frame poll | 20 ms | `exec.rs:22-23` |
| `EXIT_STDIO_GRACE` | 100 ms | `exec.rs:21` |
| scan chunk | 64 KiB | `main.rs:35` |
| `READ_CHUNK` / `WRITE_CHUNK` / depth | 256 KiB / 512 KiB / 8 in flight | `src/remote-env.ts:36-40` |
| `readFile` chunks / max | 512 KiB known, 64 KiB unknown / 2^31−1 B | `remote-env.ts:33-46` |
| `MAX_TIMEOUT_MS` | 2^31−1 | `remote-env.ts:33-46` |
| SSH / upload timeout, `ServerAliveInterval` | 60 s / 300 s / 15 | `src/ssh.ts:103,122-124` |
| watch debounce / FSEvents settle / poll / max dirs | 50 ms / 500 ms / 2000 ms / 10 000 | `daemon/src/watch.rs:21-31`, `durable/src/env/node-watch.ts:46-74` |
| watch hash | files ≤256 KiB modified within 5 s | `node-watch.ts:46-74` |
| untrusted fs magic list | 12 (NFS, SMB, CIFS, FUSE, 9P, Lustre, GPFS, Ceph, AFS, sdcardfs…) | `watch.rs:33-49` |
| watch reconnect | 1 s → 30 s | `src/watch.ts:16-17` |
| pre-sync garbage cap | 1 MiB | `connection.ts:327-342` |

## Evolution
- 2026-06-03 `86314bf38` containerization guide + Gondolin example (#5356) — tool-only isolation precedent.
- 2026-10-04 `4748c627a` (bounded readers, argv exec, env conformance), `cdf79797b` (windowed output with counted skips), `a19c09d9b` (bounded read, StreamDecoder fix), `a84510819` (`FileSystem.watch`), `864777ba6` (Windows polls), `1543dd8f6` (Windows env tests) — Durable env hardened for non-local envs.
- 2026-10-05 (all in one day, 13 commits): `ba03e03f2` pi-env created (Rust daemon, `RemoteExecutionEnv`, `Connection`; client-side `PollingWatcher` at first) → `b78e6a908` SSH bootstrap (detect, SHA-256 deploy, strict host keys, Windows) → `1965a8069` FSEvents rescan → `46d0ff936` robustness (+3154/−586: control/bulk queues, `window.rs`, native watch in daemon, 16-thread pool, `MAX_HANDLES`, poisoned chunks, `Seen` liveness, session ids) → `ed94330a2` hardened SSH, `pi-env-<sha256>` naming → `956e81504` Windows `ERROR_DIRECTORY`→`ENOTDIR` → `97a600395` Windows file identity → `4bf5a6bc5` garbled-probe detection → `6fa21f2a3` timeouts → `9a193ff7e`/`b7dfc049e` Windows upload → `68c22123b` powershell tool → `031b24aa6` lazy connections → `5b3189647` zsh login-shell test → `7c10bd433` release v1.0.4.

## Evidence commits
`ba03e03f2` `b78e6a908` `46d0ff936` `ed94330a2` `956e81504` `97a600395` `4bf5a6bc5` `6fa21f2a3` `9a193ff7e` `b7dfc049e` `031b24aa6` `7c10bd433` `1965a8069` `864777ba6` `cdf79797b` `4748c627a` `a84510819`

## Quirks
- Token is a sync marker, not authentication; trust anchor = SSH host key.
- Who constructs `RemoteExecutionEnv` in production is unknown (unverified; closed-source host suspected).
- Is skip accounting tested when `window` is ignored by `NodeExecutionEnv`? (`cdf79797b` mentions property tests; open).
- Stable pi's remote option is still the `ssh.ts` example extension (per-call `ssh`), not pi-env.

## Failures
[[bulk-output-starves-control-messages]] · [[stale-handle-reaches-new-daemon]] · [[liveness-timeout-counts-frames-not-bytes]] · [[remote-bootstrap-hang]] · [[remote-platform-misdetection]] · [[remote-errno-parity-drift]] · [[file-watch-platform-gaps]] · [[remote-binary-trusted-by-version-name]]
