---
type: group
group: 07-safety
---
What the harness does to limit damage from model actions, repo-supplied inputs, its own supply chain, remote hosts and secrets — permissions, trust, isolation, process containment.

## Concepts
- [[tool-call-gate]] — Pre-execution hook that can rewrite args or block a tool call; fail-closed on handler error; home of opt-in permission policies.
- [[tool-safety-annotations]] — Declarative unverified hints (readOnly/destructive/idempotent/openWorld) that policy plugins read.
- [[project-trust-gate]] — Remembered per-directory trust decision gating repo-supplied executable/behavior-changing config (not context files).
- [[process-tree-kill]] — Commands in their own process group/job; kill the whole tree on abort/timeout/shutdown; remote dead-man switch.
- [[tool-only-isolation]] — Agent stays on host, tool I/O routed into VM/container/remote; vs whole-process isolation.
- [[supply-chain-pinning]] — Exact pins, installer lockfile, `--ignore-scripts`, lifecycle-script allowlist, content-addressed remote binaries.
- [[remote-host-trust]] — Pinned app-owned host keys, no forwarding, transport-layer auth, sanitized errors at remote boundaries.
- [[secret-handling]] — Credentials at rest (0600), config indirection, redaction in bug reports/diagnostics; stdout protocol guard is not safety.

## Absences
[[no-permission-prompts]] · [[no-sandbox]] · [[no-cwd-confinement]] · [[no-prompt-injection-defense]]

Failures: [[Safety Failures]]
