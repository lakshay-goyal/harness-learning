---
type: concept
stage: permissions
tier: variant
aliases: [SandboxPolicy, sandbox_mode, read-only, workspace-write, danger-full-access, external-sandbox, SandboxType, PermissionProfile, FileSystemSandboxPolicy, NetworkSandboxPolicy, SandboxManager, SandboxablePreference, Seatbelt, sandbox-exec, bubblewrap, bwrap, codex-linux-sandbox, Landlock, seccomp, WindowsRestrictedToken, CodexSandboxOffline, WindowsMxc, kernel-sandbox-per-command]
harnesses: [codex]
---
Every model-launched process is wrapped in a kernel-enforced OS sandbox (one backend per OS) whose filesystem and network rights are derived from one portable policy object; the sandbox, not the prompt or the tool layer, is the trust boundary.

## Why
- Approval prompts and command classifiers only see the command text; a shell script, `make`, or a test runner executes arbitrary code the gate never saw. Only the kernel sees every `open`/`connect`.
- Without it the trust boundary is the OS user ([[no-sandbox]]), so a prompt-injected model can read `~/.ssh`, exfiltrate over the network, or write `.git/hooks` that later run with full user rights ([[agent-writes-its-own-escalation-config]]).
- A sandbox lets the harness auto-run most commands (no prompt fatigue) and reserve approvals for the commands that actually hit the boundary ([[sandbox-escalation-retry]]).
- Path-based policies and deny-lists are full of holes unless hardened per OS: rename/symlink races ([[sandbox-path-binding-races]]), alternate syscalls ([[sandbox-side-channel-syscalls]]), symlinked roots ([[symlinked-roots-escape-sandbox-policy]]).

## Design space
- **No sandbox, OS user is the boundary** — ✔ pi ([[no-sandbox]]); containment left to [[tool-only-isolation]] patterns.
- **Policy shape**: coarse modes (read-only / workspace-write / full access) vs. per-path read/write/none profiles with globs. ✔ codex has both: legacy `SandboxPolicy` modes + newer `PermissionProfile` → `FileSystemSandboxPolicy` + `NetworkSandboxPolicy`.
- **Network**: on/off flag vs. routed through a harness proxy with per-host policy ([[egress-policy-proxy]]). ✔ codex: off by default; managed proxy when configured.
- **Backend per OS** (✔ codex):
  - macOS: Seatbelt (`/usr/bin/sandbox-exec` + generated SBPL profile, `(deny default)`).
  - Linux: bubblewrap namespaces (read-only `/`, bind-mounted writable roots) + in-process seccomp network filter; legacy Landlock.
  - Windows: restricted token + ACLs (unelevated), dedicated sandbox users + WFP firewall (elevated), Microsoft MXC container (opt-in).
- **What is sandboxed**: every tool process (✔ codex: shell / unified exec / apply_patch run under the sandbox; the harness process itself does not) vs. the whole harness process (container/VM; codex ships `codex-cli/scripts/run_in_container.sh` as an external option).
- **Carve-outs inside writable roots** for files that later run with more authority ([[protected-workspace-metadata]]).
- **Per-call preference**: auto (sandbox only if policy is restrictive) / require / forbid (✔ codex `SandboxablePreference`).
- **Already sandboxed externally**: a mode that trusts an outer sandbox for disk but still honours the network flag (✔ codex `ExternalSandbox`).
- **Unavailable backend**: fail closed / fall back to prompting everything (✔ codex Windows without sandbox: default profile read-only, all unmatched commands treated as dangerous).

## Implementations
- [[codex--os-level-sandbox|codex]] — `SandboxPolicy`/`PermissionProfile` → Seatbelt (macOS), bwrap + seccomp (Linux, Landlock legacy), restricted token / elevated sandbox users + WFP / MXC (Windows); network off by default; per-OS hardening history.

## Failures
- [[agent-writes-its-own-escalation-config]]
- [[sandbox-path-binding-races]]
- [[sandbox-side-channel-syscalls]]
- [[symlinked-roots-escape-sandbox-policy]]
- [[hook-error-fails-open]] (Windows WFP setup logged non-fatally)

## Related
[[protected-workspace-metadata]] · [[approval-policy-modes]] · [[sandbox-escalation-retry]] · [[egress-policy-proxy]] · [[per-exec-interception]] · [[model-requested-permissions]] · [[permission-state-prompt]] · [[tool-only-isolation]] · [[process-tree-kill]] · [[shell-execution]] · [[no-sandbox]] · [[no-cwd-confinement]] · [[isolation-strategy]]
