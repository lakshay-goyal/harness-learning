---
type: implementation
harness: codex
concept: harness-process-hardening
commit: 622e9e3696
files: [codex-rs/process-hardening/src/lib.rs:12, codex-rs/process-hardening/src/lib.rs:28, codex-rs/responses-api-proxy/src/main.rs:6, codex-rs/voice-host/src/main.rs:49, codex-rs/linux-sandbox/src/proxy_lifecycle.rs:119]
---
[[harness-process-hardening]] in [[codex]].

## Mechanism
- `pre_main_hardening()` run via `#[ctor]` before `main` (`codex-rs/process-hardening/src/lib.rs:12-130`):
  - Linux/Android: `prctl(PR_SET_DUMPABLE, 0)` (also blocks ptrace attach), `RLIMIT_CORE=0` "for defense in depth", remove `LD_*` env vars ("Official Codex releases are MUSL-linked, which means that variables such as LD_PRELOAD are ignored anyway, but just to be sure").
  - macOS: `ptrace(PT_DENY_ATTACH)`, `RLIMIT_CORE=0`, remove `DYLD_*`.
  - FreeBSD/OpenBSD/NetBSD: `RLIMIT_CORE=0` + `LD_*`.
  - Windows: TODO no-op.
- Fail-closed: exit 5 (`prctl` failed), 6 (`PT_DENY_ATTACH` failed), 7 (`RLIMIT_CORE` failed) (`codex-rs/process-hardening/src/lib.rs:28-41`).
- Current users: `codex-responses-api-proxy` (holds the API key for unprivileged users; also mlock + zeroize, see [[codex--secret-handling]]) (`codex-rs/responses-api-proxy/src/main.rs:6`), `voice-host` (`codex-rs/voice-host/src/main.rs:49`), `disable_process_dumping()` in the Linux managed-proxy helper (`codex-rs/linux-sandbox/src/proxy_lifecycle.rs:119`). **Not** the main `codex` CLI.

## Constants
| name | value | path:line |
|---|---|---|
| hardening failure exit codes | 5 (prctl) / 6 (PT_DENY_ATTACH) / 7 (RLIMIT_CORE) | `codex-rs/process-hardening/src/lib.rs:28-41` |

## Evolution
- 2025-09-25 `d61dea6fe6` "add support for CODEX_SECURE_MODE=1 to restrict process observability (#4220)" — "the `codex` process could contain sensitive information in memory, such as API keys"; "Admittedly, a user with root privileges can defeat these safeguards".
- 2025-09-28 `43615becf0` moved into its own crate.
- 2026-01-08 `d3ff668f68` "remove existing process hardening from Codex CLI (#8951)" — stripping `LD_LIBRARY_PATH`/`DYLD_LIBRARY_PATH` broke child processes (issues #8945, #8472); it "was added not in response to a known security issue, but because it seemed like a prudent thing to do".

## Versus pi
- pi has nothing equivalent; its secret story is file modes + redaction ([[pi--secret-handling]]).
