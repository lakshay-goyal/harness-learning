---
type: concept
stage: permissions
tier: variant
aliases: [codex-network-proxy, managed network, network sandbox proxy, domains allow/deny, "mode full|limited", NetworkPolicyDecider, NetworkPolicyAmendment, x-proxy-error, credential broker, MITM CA, allow_local_binding, per-host egress allowlist]
harnesses: [codex]
---
The OS sandbox only permits traffic to a harness-owned loopback proxy, which enforces a domain allow/deny list (optionally method limits, MITM hooks, credential substitution) and turns "network on/off" into per-host policy with approvals and persistent rules.

## Why
- Binary network flags fail both ways: off breaks `npm install`/`cargo build`; on allows exfiltration to any host.
- Approvals need a *host* to show the user ("allow `pypi.org`?") — only a proxy sees it.
- Child processes inherit real credentials from the environment and can exfiltrate them; a proxy can hold the secret and inject it only on the wire to bound hosts.
- Every error path in an egress check must default to deny ([[network-policy-fail-open-paths]]), and every non-proxied kernel path must be closed ([[sandbox-side-channel-syscalls]]).

## Design space
- **No network control** — pi (no sandbox).
- **Network on/off in the sandbox** — codex default (`network_access=false`).
- **Firewall allowlist in a container** — codex `codex-cli/scripts/init_firewall.sh` (iptables + ipset) for whole-process isolation.
- **Loopback proxy enforced by the sandbox** (✔ codex managed network): Seatbelt allows only `localhost:<proxy port>`; Linux `--unshare-net` + TCP→UDS→TCP bridge; MXC loopback-only; Windows non-elevated only env-var proxy poisoning.
- **Policy**: allowlist-first + deny wins + scoped wildcards; private/link-local blocked incl. hostnames resolving there; limited mode (GET/HEAD/OPTIONS, HTTPS via MITM).
- **Approval on miss**: decider hook may only override allowlist misses → approval prompt → persisted `network_rule` (✔ codex).
- **Credential handling**: real env credentials vs proxy-held credentials with dummy values in the child (✔ codex experimental broker) — cf. Docker Sandboxes placeholder substitution in [[tool-only-isolation]].
- **Audit**: per-decision events without full URLs (✔ codex).

## Implementations
- [[codex--egress-policy-proxy|codex]] — `codex-network-proxy` (HTTP 127.0.0.1:3128, SOCKS5 :8081); domain allow/deny; limited mode; MITM CA in memory; `x-proxy-error`; decider → approval → `network_rule`; credential broker.

## Failures
- [[network-policy-fail-open-paths]]
- [[sandbox-side-channel-syscalls]]
- [[sandbox-network-error-not-escalated]]

## Related
[[os-level-sandbox]] · [[sandbox-escalation-retry]] · [[command-rule-policy]] · [[approval-policy-modes]] · [[secret-handling]] · [[tool-only-isolation]] · [[web-tools]] · [[isolation-strategy]]
