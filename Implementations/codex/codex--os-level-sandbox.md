---
type: implementation
harness: codex
concept: os-level-sandbox
commit: 622e9e3696
files: [codex-rs/protocol/src/protocol.rs:1093, codex-rs/protocol/src/protocol.rs:1114, codex-rs/protocol/src/sandbox.rs:29, codex-rs/sandboxing/src/manager.rs:44, codex-rs/sandboxing/src/manager.rs:50, codex-rs/sandboxing/src/manager.rs:313, codex-rs/sandboxing/src/seatbelt.rs:21, codex-rs/sandboxing/src/seatbelt.rs:62, codex-rs/sandboxing/src/seatbelt_base_policy.sbpl:1, codex-rs/linux-sandbox/README.md:10, codex-rs/linux-sandbox/src/landlock.rs:43, codex-rs/windows-sandbox-rs/src/token.rs:42, codex-rs/windows-sandbox-rs/src/setup.rs:53, codex-rs/windows-sandbox-rs/src/env.rs:126, codex-rs/windows-sandbox-service/src/ipc.rs:45, codex-rs/mxc-sandbox/README.md:1, codex-rs/core/src/config/permissions.rs:50]
---
[[os-level-sandbox]] in [[codex]].

## Mechanism

### Policy objects (portable layer)
- `SandboxPolicy` = `DangerFullAccess` | `ReadOnly{network_access}` | `ExternalSandbox{network_access}` (process already inside an outer sandbox: full disk, network flag honoured) | `WorkspaceWrite{writable_roots, network_access, exclude_tmpdir_env_var, exclude_slash_tmp}` (`codex-rs/protocol/src/protocol.rs:1114-1158`).
- Network flag defaults `false` in ReadOnly and WorkspaceWrite (`codex-rs/protocol/src/protocol.rs:1122-1125`, `codex-rs/protocol/src/protocol.rs:1144-1147`); `NetworkAccess` enum default `Restricted` (`codex-rs/protocol/src/protocol.rs:1093-1099`).
- WorkspaceWrite writable roots = cwd + `writable_roots` + `$TMPDIR` + `/tmp` unless excluded (doc comments `codex-rs/protocol/src/protocol.rs:1137-1157`). Inside each root, [[protected-workspace-metadata]] stays read-only.
- Richer layer: `PermissionProfile` → `(FileSystemSandboxPolicy, NetworkSandboxPolicy)` via `to_runtime_permissions()`; per-path read / write / none entries incl. globs (`"**/*.env" = "none"`). `SandboxPolicy` survives as legacy/compat (`codex-rs/sandboxing/src/manager.rs:342-350`; `compatibility_sandbox_policy_for_permission_profile` at `codex-rs/sandboxing/src/manager.rs:756`).
- Default profile: workspace-write if the project trust decision was made (trusted or untrusted), read-only if never decided or if the Windows sandbox is disabled (`codex-rs/core/src/config/permissions.rs:50-61`). See [[project-trust-gate]].
- CLI: `--sandbox/-s <mode>`; `--dangerously-bypass-approvals-and-sandbox` (alias `yolo`) "EXTREMELY DANGEROUS. Intended solely for running in environments that are externally sandboxed" (`codex-rs/utils/cli/src/shared_options.rs:40-63`). Debug surface `codex sandbox` (`codex-rs/linux-sandbox/README.md:111`; was `codex debug seatbelt|landlock`, `e79549f039`).

### Backend selection
- `get_platform_sandbox()`: macOS → `MacosSeatbelt`, Linux → `LinuxSeccomp`, Windows → `WindowsRestrictedToken` only when the Windows sandbox is enabled, else `None` (`codex-rs/sandboxing/src/manager.rs:50-64`). `select_initial` prefers `WindowsMxc` when explicitly selected (`codex-rs/sandboxing/src/manager.rs:313-331`).
- Per-call `SandboxablePreference::{Auto, Require, Forbid}` (`codex-rs/sandboxing/src/manager.rs:44-48`); `Auto` sandboxes only if the profile is actually restrictive or managed network requirements exist (`should_sandbox`, `codex-rs/sandboxing/src/manager.rs:334-353`).
- Metric tags `SandboxType`: none / seatbelt / seccomp / windows_sandbox / windows_mxc (`codex-rs/protocol/src/sandbox.rs:29-48`). `codex-rs/core/src/sandbox_tags.rs` labels are diagnostics only: "must never be used to authorize filesystem access" (header).
- Windows level mapping: MXC preserved; `Disabled` → None; `RestrictedToken | Elevated` → `WindowsRestrictedToken` (`effective_windows_sandbox_type`, `codex-rs/protocol/src/sandbox.rs:50-60`).
- Orchestration: approval → select sandbox → attempt → escalate on denial lives in `codex-rs/core/src/tools/orchestrator.rs` ([[sandbox-escalation-retry]]).

### macOS — Seatbelt
- Runs absolute `/usr/bin/sandbox-exec` (`MACOS_PATH_TO_SEATBELT_EXECUTABLE`, `codex-rs/sandboxing/src/seatbelt.rs:62`) — no PATH lookup.
- Profile assembled from embedded files (`codex-rs/sandboxing/src/seatbelt.rs:21-28`): `seatbelt_base_policy.sbpl` (116 lines), `seatbelt_network_policy.sbpl` (31), `seatbelt_preferences_policy.sbpl` (8), `seatbelt_read_only_platform_defaults.sbpl` (188), plus inline TLS trust rule `(allow mach-lookup (global-name "com.apple.TrustEvaluationAgent"))` ("System libcurl needs this service for TLS").
- Base policy "inspired by Chrome's sandbox policy": `(deny default)`, allow `process-exec`/`process-fork` (children inherit), signals only within the sandbox, allow-listed sysctls (e.g. `kern.sysv.semmns` for Python `ProcessPoolExecutor`), POSIX sem/shm for Python multiprocessing / libomp, pty access (`codex-rs/sandboxing/src/seatbelt_base_policy.sbpl:1-116`).
- Paths passed as `(param "WRITABLE_ROOT_n")`, never string-interpolated into SBPL (`codex-rs/sandboxing/src/seatbelt.rs:513-517`).
- Writable roots get `(deny file-write-unlink …)` on the root itself: "A sandboxed process must not be able to replace an authority boundary that will be reused to build the next sandbox policy" (`codex-rs/sandboxing/src/seatbelt.rs:518-521`).
- Protected subpaths: both `(require-not (literal …))` and `(require-not (subpath …))` — "`subpath` alone leaves a gap for first-time creation of the protected directory itself, such as `mkdir .codex`" (`codex-rs/sandboxing/src/seatbelt.rs:565-576`); metadata names via regex `^{root}/{name}(/.*)?$` (`codex-rs/sandboxing/src/seatbelt.rs:592-604`); ancestors of read-only paths protected against rename (`codex-rs/sandboxing/src/seatbelt.rs:910-934`).
- Full-disk-write case: `(allow file-write* (regex #"^/"))` — comment "Allegedly, this is more permissive than `(allow file-write*)`" (`codex-rs/sandboxing/src/seatbelt.rs:944-949`).
- Network: off + no proxy ⇒ no network allow clauses at all. With proxy env / managed network: only `localhost:<proxy port>` outbound + optional local binding; proxy configured but no loopback port inferable ⇒ "Fail closed to avoid silently widening network access" (`codex-rs/sandboxing/src/seatbelt.rs:319-380`, comments at `:358`, `:364`). Unix-socket proxying macOS-only via `x-unix-socket` ([[egress-policy-proxy]]).
- Restricted policies block mutating `fcntl`s `F_MAKECOMPRESSED`/`F_TRANSFEREXTENTS` (`04e4d2b40f`).

### Linux — bubblewrap + seccomp (Landlock legacy)
- Helper `codex-linux-sandbox` is the same multitool binary dispatched on arg0 (`codex-rs/linux-sandbox/README.md:1-8`; `codex-rs/arg0/src/lib.rs:21-25`). `codex-rs/bwrap/` = thin standalone build of vendored C bubblewrap (`codex-rs/bwrap/src/main.rs`, `codex-rs/bwrap/build.rs`).
- bwrap resolution: first `bwrap` on PATH *outside the cwd* (a repo cannot plant one), too-old bwrap without `--argv0` → compat path, missing → bundled `codex-resources/bwrap` + startup warning; userns unavailable → startup warning; WSL1 rejected before invoking bwrap, WSL2 normal (`codex-rs/linux-sandbox/README.md:10-39`).
- Filesystem (`codex-rs/linux-sandbox/README.md:40-95`):
  - `--ro-bind / /`; writable roots `--bind <root> <root>`; protected subpaths (`.git`, resolved `gitdir:`, `.codex`) re-applied `--ro-bind`.
  - Split policies applied in path-specificity order: `/repo = write`, `/repo/a = none`, `/repo/a/b = write` works.
  - Unreadable globs pre-expanded with `rg --files --hidden --no-ignore --glob <pattern> -- <root>` (fallback internal globset walker; any other rg failure aborts sandbox construction); `glob_scan_max_depth` per profile.
  - Symlink-in-path and missing protected paths masked by mounting `/dev/null` on the symlink / first missing component.
  - `--unshare-user`, `--unshare-pid` by default, `--unshare-net` when network restricted without proxy routing, fresh `--proc /proc` (falls back to inherited `/proc` if denied).
  - Legacy Landlock rejected for filesystem-restricted execution: "it cannot isolate app-server Unix sockets" (`codex-rs/linux-sandbox/README.md:40-42`).
- PID-namespace inherit only via trusted startup flag `codex exec-server --linux-sandbox-pid-namespace=inherit`; "repository config and command environment variables cannot enable it" (`codex-rs/linux-sandbox/README.md:96-108`).
- In-process seccomp (helper thread): `apply_permission_profile_to_current_thread` sets `PR_SET_NO_NEW_PRIVS` only when seccomp or legacy Landlock is needed because "Many `bwrap` deployments rely on setuid" (`codex-rs/linux-sandbox/src/landlock.rs:43-96`, comment `:66-69`).
- Seccomp modes `Restricted | ProxyRouted | VmSocketRestricted` (`codex-rs/linux-sandbox/src/landlock.rs:99-104`); managed network stays fail-closed even under DangerFullAccess (`codex-rs/linux-sandbox/src/landlock.rs:106-113`).
- Denied syscalls (`codex-rs/linux-sandbox/src/landlock.rs:183-231`): non-VM modes `ptrace`, `process_vm_readv/writev`; all modes `io_uring_setup/enter/register` ("io_uring can create AF_VSOCK sockets without a socket() syscall"); restricted mode `connect/accept/accept4/bind/listen/getpeername/getsockname/shutdown/sendto/sendmmsg/recvmmsg/getsockopt/setsockopt`, `socket`/`socketpair` only `AF_UNIX`. `recvfrom` deliberately left allowed (commented out, `:216`). Default action Allow, match → `EPERM`; x86_64/aarch64 only (`codex-rs/linux-sandbox/src/landlock.rs:280-297`). AF_VSOCK denied even with network allowed, `/run/WSL` masked (`codex-rs/linux-sandbox/src/landlock.rs:53-62`, `f71543813f`).
- Managed proxy mode: `--unshare-net` + internal TCP→UDS→TCP bridge so tool traffic reaches only proxy endpoints; after the bridge is live seccomp blocks new `AF_UNIX`/`socketpair` (`codex-rs/linux-sandbox/README.md:85-89`; `codex-rs/linux-sandbox/src/proxy_routing.rs`). Proxy helper sets `PR_SET_PDEATHSIG` and disables dumping (`codex-rs/linux-sandbox/src/proxy_lifecycle.rs:119-123`).
- Legacy Landlock: ABI V5, read-only `/`, rw `/dev/null` + writable roots; refuses restricted-read policies ("Restricted read-only access is not supported by the legacy Linux Landlock filesystem backend") (`codex-rs/linux-sandbox/src/landlock.rs:79-84`, `:147-170`). Only override: `features.use_legacy_landlock`.

### Windows — restricted token, elevated sandbox users, MXC
- **Unelevated restricted token**: `CreateRestrictedToken` with `DISABLE_MAX_PRIVILEGE | LUA_TOKEN | WRITE_RESTRICTED` (`codex-rs/windows-sandbox-rs/src/token.rs:42-44`, call `:499`); restricting SIDs in exact order capability SIDs…, extra, logon SID, Everyone (`codex-rs/windows-sandbox-rs/src/token.rs:480-497`). Write-restricted only: writes need an ACL grant to a capability SID; reads broadly allowed. Per-workspace capability SIDs since `aabe0f259c` (before, one SID could write every workspace). Default DACL so the token can read pipes (`6395430220`).
- **Elevated**: dedicated local users `CodexSandboxOffline` / `CodexSandboxOnline` (`codex-rs/windows-sandbox-rs/src/setup.rs:53-54`), group `CodexSandboxUsers` (`codex-rs/windows-sandbox-rs/src/winutil.rs:29`); offline user firewalled by persistent WFP filters keyed on `FWPM_CONDITION_ALE_USER_ID` (`install_wfp_filters_for_account`, `codex-rs/windows-sandbox-rs/src/wfp.rs:84`); commands run via `codex-command-runner.exe` over framed IPC (`codex-rs/windows-sandbox-rs/src/helper_materialization.rs:24`; `95bdea93d2`). Setup needs UAC once.
- **Unelevated "no network" is env poisoning, not a kernel block**: `apply_no_network_to_env` sets `SBX_NONET_ACTIVE=1`, `HTTP(S)_PROXY`/`ALL_PROXY`/`GIT_HTTP(S)_PROXY=http://127.0.0.1:9` (discard port), `PIP_NO_INDEX=1`, `NPM_CONFIG_OFFLINE=true`, `CARGO_NET_OFFLINE=true`, `GIT_SSH_COMMAND="cmd /c exit 1"` (`codex-rs/windows-sandbox-rs/src/env.rs:126-160`) plus a `denybin` stub dir (`codex-rs/windows-sandbox-rs/src/env.rs:105`). Raw sockets bypass it; kernel-level egress block only in elevated (WFP) and MXC.
- Private desktop `CodexSandboxDesktop-*` (`codex-rs/windows-sandbox-rs/src/desktop.rs:67`) instead of `Winsta0\Default` (`6b3d82daca`; briefly disabled `2cc4ee413f`; opt-out key `windows.sandbox_private_desktop` removed 2026-09-18 `a633ebc124` "Always use private desktops for legacy Windows sandboxes" — strict config now errors with "Remove windows.sandbox_private_desktop", `codex-rs/config/src/strict_config.rs:304-305`). `[windows]` keys at HEAD: `sandbox`, `allow_mxc` ("False blocks both explicit MXC configuration and automatic selection") (`codex-rs/config/src/types.rs:195-199`).
- ACL hygiene: world-writable directory scan + warning (`871d442b8e`) and deny ACEs (`3f92ad4190`); `.git` file entries denied under writable roots (`f2de920185`); deny-read via deny ACEs on both lexical and canonical paths (`codex-rs/windows-sandbox-rs/src/deny_read_acl.rs:10-13`; managed deny-read enforced `848cbad7f4`); directory opens refuse reparse points (`codex-rs/windows-sandbox-rs/src/no_reparse_dir.rs:1-3`); sandbox user profile dir hidden best-effort (`codex-rs/windows-sandbox-rs/src/hide_users.rs:78-81`).
- **`codex-rs/windows-sandbox-service/`**: SCM service `CodexSandboxService` performing privileged provisioning (users / ACLs / WFP) over an authenticated named pipe; IPC request ≤ 4096 B, response message ≤ 512 B, idle timeout 5 s (`codex-rs/windows-sandbox-service/src/ipc.rs:45-48`). Added `501931b399` (body: keep provisioning IPC disabled until authenticated request handling exists), hardened `add870a4bf`.
- **MXC** (`codex-rs/mxc-sandbox/README.md`): routes the command into Microsoft MXC `BaseContainerRunner`; "never invokes MXC's AppContainer dispatcher, edits host ACLs, creates sandbox users, runs setup, or requests elevation" (`:3-7`); deny paths require `PSE_SUPPORT_FS_DENY` else fail before launch (`:13-17`); child created suspended, kill-on-close job assigned before resume; managed network = IPv4/IPv6 loopback only, direct non-loopback egress denied (`:19-25`); `windows.sandbox = "mxc"` strict, `features.prefer_mxc` default off, "Command failures never trigger backend fallback" (`:31-34`); launcher env chunks ≤ 4096 B, payload ≤ 1,000,000 B (`:48-50`); `allow_local_binding` defaults `true` under MXC and `false` is a config error (`:60-67`); upstream runner kills descendants on foreground exit, unlike the other Windows backends (`:75-79`).
- No Windows sandbox enabled ⇒ default profile read-only *policy shape* and unmatched commands treated as dangerous: "there is no platform sandbox to enforce the boundary" (`codex-rs/core/src/config/permissions.rs:55-61`; `codex-rs/core/src/exec_policy.rs:787-806`).

### Whole-process option (outside the sandbox crates)
- `codex-cli/scripts/run_in_container.sh`: Docker with `--cap-add=NET_ADMIN --cap-add=NET_RAW`, root-only `init_firewall.sh` (iptables default DROP + ipset allowlist from `/etc/codex/allowed_domains.txt`, default `api.openai.com`), then runs `codex --sandbox workspace-write --ask-for-approval on-request` (`codex-cli/scripts/run_in_container.sh:60-95`; `codex-cli/scripts/init_firewall.sh:5-17`, `:79-94`). Present since the initial commit `59a180ddec`.

## Constants
| name | value | path:line |
|---|---|---|
| default network in sandbox | off (`network_access=false`, `NetworkAccess::Restricted`) | `codex-rs/protocol/src/protocol.rs:1093-1147` |
| macOS sandbox executable | `/usr/bin/sandbox-exec` | `codex-rs/sandboxing/src/seatbelt.rs:62` |
| SBPL base / network / prefs / read-only defaults | 116 / 31 / 8 / 188 lines | `codex-rs/sandboxing/src/seatbelt.rs:21-28` |
| Landlock ABI (legacy) | V5 | `codex-rs/linux-sandbox/src/landlock.rs:150` |
| seccomp deny (all modes) | `io_uring_setup/enter/register` | `codex-rs/linux-sandbox/src/landlock.rs:195-199` |
| seccomp action | default Allow, matched → `EPERM` | `codex-rs/linux-sandbox/src/landlock.rs:280-284` |
| restricted token flags | `DISABLE_MAX_PRIVILEGE \| LUA_TOKEN \| WRITE_RESTRICTED` (0x01/0x04/0x08) | `codex-rs/windows-sandbox-rs/src/token.rs:42-44` |
| Windows sandbox users | `CodexSandboxOffline` / `CodexSandboxOnline` / group `CodexSandboxUsers` | `codex-rs/windows-sandbox-rs/src/setup.rs:53-54`; `codex-rs/windows-sandbox-rs/src/winutil.rs:29` |
| Windows no-network sink | `http://127.0.0.1:9`, `GIT_SSH_COMMAND=cmd /c exit 1` | `codex-rs/windows-sandbox-rs/src/env.rs:126-160` |
| private desktop prefix | `CodexSandboxDesktop-` | `codex-rs/windows-sandbox-rs/src/desktop.rs:67` |
| launch env transport | chunk 16 KiB, max 16 MiB (`CODEX_SANDBOX_LAUNCH_*`) | `codex-rs/windows-sandbox-rs/src/environment_transport.rs:11-16` |
| IPC frame / poll | 8 MiB / 5 ms | `codex-rs/windows-sandbox-rs/src/framed_io.rs:17-18` |
| sandbox logs kept / preview | 90 files / 200 chars | `codex-rs/windows-sandbox-rs/src/logging.rs:15-18` |
| service IPC limits | request 4096 B, response 512 B, idle 5 s | `codex-rs/windows-sandbox-service/src/ipc.rs:45-48` |
| MXC env chunk / payload | 4096 B / 1,000,000 B | `codex-rs/mxc-sandbox/README.md:48-50` |

## Evolution
- 2025-04-16 `59a180ddec` initial commit (TS CLI) already with Seatbelt + container firewall scripts; 2025-04-24 `31d0d7a305` Rust import with Seatbelt and Landlock; 2025-04-27 `e9d16d3c2b` check `sandbox-exec` availability; 2025-04-28 `e79549f039` `debug landlock` next to `debug seatbelt`.
- 2025-05-23 `89ef4efdcf` spawn under seccomp/landlock overhauled.
- 2025-08-01 `80555d4ff2` `.git` read-only on Seatbelt (start of [[protected-workspace-metadata]]).
- 2025-10-30 `87cce88f48` "Windows Sandbox - Alpha version"; 2025-11-06 `871d442b8e` / 2025-11-20 `3f92ad4190` world-writable dirs; 2025-12-09..12 `fc4249313b`/`13c0919bff`/`3e81ed4b91`/`677732ff65` "Elevated Sandbox 1..4"; 2025-12-18 `6395430220` default DACL.
- 2026-01-14 `2259031d64` Landlock-only fallback when user namespaces unavailable + early `PR_SET_NO_NEW_PRIVS`; 2026-01-20 `f2de920185` deny `.git` file entries (Windows).
- 2026-02-02 `f956cc2a02` vendor bubblewrap via FFI; 2026-02-03 `aabe0f259c` per-workspace capability SIDs; 2026-02-04 `ae4de43ccc` bwrap support; 2026-02-06 `8896ca0ee6` block io_uring.
- 2026-03-06 `b52c18e414` effective file access from filesystem policies (`PermissionProfile` era); 2026-03-11 `04892b4ceb` bubblewrap default on Linux; 2026-03-13 `6b3d82daca` private desktop (2026-03-17 `2cc4ee413f` temporarily disabled); 2026-03-17 `95bdea93d2` framed IPC for elevated runner; 2026-03-26 `b6050b42ae` bwrap from trusted PATH entry.
- 2026-04-12 `cb870a169a` reject WSL1; 2026-04-30 `8121710ffe` WFP filters for offline sandbox user (failures logged non-fatally).
- 2026-08-11 `1dac3d9ca0` fail closed on unsafe unreadable globs; 2026-08-13 `779e9114ae` reap orphaned processes in Linux sandboxes; 2026-08-14 `848cbad7f4` managed deny-read in Windows sandbox; 2026-08-19 `52e387daca` / `f6950546e5`, 2026-08-20 `02de49f718` Seatbelt rename / anchor / path-binding hardening.
- 2026-09-02 `501931b399` / `add870a4bf` Windows sandbox service; 2026-09-04 `60888d0868` MXC adapter, 2026-09-08 `ce254df05a` canonical permission translation for MXC; 2026-09-09 `f71543813f` block WSL interop escapes; 2026-09-18 `04e4d2b40f` block mutating fcntls.
- 2026-10-06 `ccde2fc8b7` protect ripgrep lookup during Linux sandbox construction.

## Quirks
- Linux enum name `LinuxSeccomp` though filesystem isolation is bubblewrap; seccomp is only the network/syscall layer.
- Windows default is *no* sandbox unless `windows.sandbox = "elevated" | "unelevated" | "mxc"` is set (legacy feature flags `WindowsSandbox` / `WindowsSandboxElevated` default off) (`codex-rs/core/src/windows_sandbox.rs:62-79`); the harness compensates by prompting for everything (read-only profile + dangerous-by-default).
- Seccomp filter is a denylist (default Allow) — every new socket-creating path (io_uring, AF_VSOCK) needed its own fix ([[sandbox-side-channel-syscalls]]).
- `docs/sandbox.md` is a 3-line stub pointing to developers.openai.com (`docs/sandbox.md:1-3`).

## Versus pi
- pi ships no sandbox ([[no-sandbox]]); [[pi--tool-only-isolation]] documents container / VM / bash-only sandbox examples as opt-ins. Codex makes a per-command kernel sandbox the default trust boundary. See [[isolation-strategy]].
