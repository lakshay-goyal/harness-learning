---
type: implementation
harness: codex
concept: egress-policy-proxy
commit: 622e9e3696
files: [codex-rs/network-proxy/README.md:5, codex-rs/network-proxy/README.md:237, codex-rs/network-proxy/src/credential_broker.rs:43, codex-rs/network-proxy/src/mitm.rs:94, codex-rs/network-proxy/src/attribution.rs:14, codex-rs/sandboxing/src/seatbelt.rs:319, codex-rs/linux-sandbox/README.md:85, codex-rs/core/src/tools/orchestrator.rs:474, codex-rs/core/src/network_policy_decision.rs:74, codex-rs/execpolicy/src/amend.rs:85]
---
[[egress-policy-proxy]] in [[codex]].

## Mechanism
- Listeners: HTTP proxy default `127.0.0.1:3128`, SOCKS5 `127.0.0.1:8081` (enabled by default); Windows preferred ranges 3128-3159 / 8081-8112 (`codex-rs/network-proxy/README.md:5-9`). Config under `permissions.<profile>.network`; `mode = "full"` default or `"limited"`; `enabled=false` → proxy no-ops (`codex-rs/network-proxy/README.md:27-39`).
- Sandbox routing (only proxy reachable):
  - Seatbelt: only `localhost:<proxy port>` outbound; proxy config without inferable loopback port → fail closed (`codex-rs/sandboxing/src/seatbelt.rs:319-380`).
  - Linux: `--unshare-net` + internal TCP→UDS→TCP bridge; seccomp `ProxyRouted` mode blocks new AF_UNIX/socketpair once the bridge is live (`codex-rs/linux-sandbox/README.md:85-89`).
  - MXC: IPv4/IPv6 loopback only; `allow_local_binding` defaults true there (`codex-rs/mxc-sandbox/README.md:23-25`, `:60-67`).
  - Windows non-elevated: env-var poisoning only (`codex-rs/windows-sandbox-rs/src/env.rs:126-160`) — see [[codex--os-level-sandbox]].
  - Spawned commands get `HTTP(S)_PROXY`, `WS(S)_PROXY`, `ALL_PROXY=socks5h://…` and, with MITM, CA bundle env vars pointing at immutable public files under `$CODEX_HOME/proxy/` (`codex-rs/network-proxy/README.md:40-43`, `:116-130`).
- Policy semantics (`README.md` "Security notes", `:237-268`): allowlist-first (no `allow` entries → everything blocked); exact hosts, `*.x` = subdomains only, `**.x` = apex + subdomains, `?` = one char incl. dot, global `*` rejected unless explicit; deny wins; `allow_local_binding=false` blocks loopback/private/link-local and hostnames resolving to private IPs even if allowlisted (best-effort DNS); limited mode = GET/HEAD/OPTIONS only, HTTPS CONNECT and SOCKS5 :443 need MITM else blocked, SOCKS5 UDP blocked; non-loopback listener binds clamped unless `dangerously_allow_non_loopback_proxy`; unix-socket proxying (macOS only, `x-unix-socket` header) forces all listeners to loopback; `dangerously_allow_all_unix_sockets` bypasses the socket allowlist. Stated limitation: DNS rebinding not fully preventable — "enforce network egress at a lower layer too".
- Blocks: HTTP `403` + `x-proxy-error: blocked-by-allowlist | blocked-by-denylist | blocked-by-method-policy | blocked-by-policy` (`codex-rs/network-proxy/README.md:135-141`).
- Decider hook `NetworkPolicyDecider` receives `command` / `exec_policy_hint`; may override only `not_allowed` (allowlist miss), never `denied` or `not_allowed_local` (`codex-rs/network-proxy/README.md:181-189`). Core maps a miss into an approval with retry reason `Network access to "<host>" is blocked by policy.` (`codex-rs/core/src/tools/orchestrator.rs:474-481`); user may persist `NetworkPolicyAmendment` allow/deny as execpolicy `network_rule` (`codex-rs/execpolicy/src/amend.rs:85`; `codex-rs/core/src/network_policy_decision.rs:74`); reviewer kind `NetworkAccess` ([[codex--llm-approval-reviewer]]). Deferred approvals: dropping an unfinished owner "fails its existing waiters closed" (`codex-rs/core/src/tools/network_approval.rs` header); pending reviews after process completion handled (`8e85265c39`).
- Audit: OTEL `codex.network_proxy.policy_decision` with scope/decision/source/reason/host/port/method; "Audit events intentionally avoid logging full URL/path/query data" (`codex-rs/network-proxy/README.md:191-228`).
- Credential broker (experimental): real credentials discovered at child setup held only in proxy memory; child gets "shaped dummy values"; proxy substitutes on TLS traffic to bound hosts (GitHub, GHE, OpenAI) (`989f55defa`: "Codex child processes can inherit injectable local credentials directly, which lets commands read and exfiltrate the real values"; hardened `eea28321ad`). Env keys `CODEX_NETWORK_PROXY_CREDENTIAL_BROKER_ACTIVE`, `CODEX_NETWORK_PROXY_BROKERED_CREDENTIALS` (`codex-rs/network-proxy/src/credential_broker.rs:43-44`).
- MITM: CA private key in proxy memory only (`c5a9a95ab6`); body inspection off, cap 4096 B (`codex-rs/network-proxy/src/mitm.rs:94-95`).
- Attribution of connections to commands: frame magic `\0CDXPXY1`, token ≤ 128 B, 3 s frame timeout, env `CODEX_NETWORK_PROXY_ATTRIBUTION` (`codex-rs/network-proxy/src/attribution.rs:14-18`).
- Escalated (unsandboxed) exec bypasses managed proxy state (`9aaa5d9358`).

## Constants
| name | value | path:line |
|---|---|---|
| proxy default ports | HTTP 127.0.0.1:3128, SOCKS5 127.0.0.1:8081 | `codex-rs/network-proxy/README.md:5-6` |
| Windows port ranges | 3128-3159 / 8081-8112 | `codex-rs/network-proxy/README.md:8-9` |
| limited-mode methods | GET, HEAD, OPTIONS | `codex-rs/network-proxy/README.md:143` |
| MITM body cap | 4096 B, inspection off | `codex-rs/network-proxy/src/mitm.rs:94-95` |
| MIN_EMBEDDED_CREDENTIAL_LENGTH | 16 (alias marker `@alias:`) | `codex-rs/network-proxy/src/credential_broker.rs:45-46` |
| MAX_CONFIG_BYTES | 1 MiB | `codex-rs/network-proxy/src/main.rs:22` |
| attribution frame | magic `\0CDXPXY1`, token ≤ 128, timeout 3 s | `codex-rs/network-proxy/src/attribution.rs:14-18` |
| brokered tunnel | prefix timeout 250 ms, request line ≤ 8192 | `codex-rs/network-proxy/src/brokered_tunnel.rs:39-40` |

## Evolution
- 2026-01-23 `77222492f9` "introducing a network sandbox proxy (#8442)"; 2026-01-27 `877b76bb9d` SOCKS5 with policy enforcement.
- 2026-02-23 `c3048ff90a` persist network approvals in execpolicy.
- 2026-03-26 `aea82c63ea` fail closed on DNS lookup errors.
- 2026-04-25 `9aaa5d9358` bypass managed network for escalated exec; 2026-04-28 `3afb185a4f` tighten bypass defaults (NO_PROXY no longer covers 169.254.0.0/16 / cloud metadata).
- 2026-06-23 `c5a9a95ab6` MITM CA key in memory only; 2026-06-24 `989f55defa` credential broker; 2026-08-11 `eea28321ad` broker hardening.
- 2026-09-04 `8e85265c39` pending network reviews after process completion.
- 2026-10-07 `37eaae6eeb` "Require hostname authorization before proxy DNS lookups" (DNS leak + Seatbelt port-53 allowance removed).

## Versus pi
- pi has no egress control; its docs point to Docker Sandboxes with a placeholder-substituting proxy ([[pi--tool-only-isolation]]) — the codex credential broker is the same idea, local and per-command.
