---
type: implementation
harness: codex
concept: tool-only-isolation
commit: 622e9e3696
files: [codex-rs/sandboxing/src/manager.rs:50, codex-rs/protocol/src/protocol.rs:1114, codex-rs/network-proxy/src/credential_broker.rs:43, codex-rs/linux-sandbox/README.md:96, codex-cli/scripts/run_in_container.sh:60, codex-cli/scripts/init_firewall.sh:79]
---
[[tool-only-isolation]] in [[codex]].

## Mechanism
- **Default = tool-only, local, kernel-enforced**: the harness process (model client, credentials, extensions) runs unsandboxed on the host; every model-launched command is wrapped per call by the OS sandbox ([[codex--os-level-sandbox]]). Unlike pi's opt-in examples, codex covers *all* exec paths (shell, unified exec, apply_patch) through one `SandboxManager` (`codex-rs/sandboxing/src/manager.rs:50-64`, `:334-353`).
- **Whole-process option**: `SandboxPolicy::ExternalSandbox{network_access}` = "process already inside an outer sandbox" (`codex-rs/protocol/src/protocol.rs:1114-1158`); `--dangerously-bypass-approvals-and-sandbox` "Intended solely for running in environments that are externally sandboxed"; repo ships `codex-cli/scripts/run_in_container.sh` (Docker, `NET_ADMIN`/`NET_RAW`, iptables default DROP + ipset domain allowlist, then `codex --sandbox workspace-write --ask-for-approval on-request`) (`codex-cli/scripts/run_in_container.sh:60-95`; `codex-cli/scripts/init_firewall.sh:79-94`) — container *and* inner sandbox.
- **Remote tool execution**: `codex exec-server` runs tools on a provisioned remote environment (trusted startup flags like `--linux-sandbox-pid-namespace=inherit`, `codex-rs/linux-sandbox/README.md:96-108`); remote executors keep their configured sandbox backend (`codex-rs/mxc-sandbox/README.md:33-34`); remote `apply_patch` sandboxed (`34db7e5563`); remote sandbox denials reported semantically (`9f06cf1a09`).
- **Credential-proxy substitution, locally**: experimental network-proxy credential broker gives children "shaped dummy values" and swaps real credentials in only on TLS to bound hosts (`989f55defa`; `codex-rs/network-proxy/src/credential_broker.rs:43-46`) — the Docker-Sandboxes idea applied per command on the host ([[codex--egress-policy-proxy]]).
- **Not covered**: the harness process itself, MCP servers it launches (run with user rights; unverified whether sandboxed), hooks (external processes; trust-gated not sandboxed — [[codex--project-trust-gate]]).

## Evolution
- 2025-04-16 `59a180ddec` initial commit already had Seatbelt + container firewall scripts.
- 2025-11-18 `c1391b9f94` exec-server.
- 2026-06-24 `989f55defa` credential broker.
- 2026-08-11 `34db7e5563` "Sandbox remote apply_patch operations".

## Versus pi
- [[pi--tool-only-isolation]]: no built-in sandbox; `*Operations` seams + examples (Gondolin VM, SSH, bash-only sandbox-exec/bwrap) and a containerization doc. Codex inverts the default: per-command kernel sandbox always on (macOS/Linux), container as an additional outer layer. See [[isolation-strategy]].
