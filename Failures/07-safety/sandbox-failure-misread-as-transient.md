---
type: failure
concepts: [tool-call-gate, tool-only-isolation]
harnesses: [codex]
---
**Symptom** — Under `on-request` + workspace-write, sandbox-induced DNS/registry failures (`Could not resolve host: index.crates.io`) reached the model as ordinary tool errors; it treated them as transient and never requested escalation (`6046ca19ba` body).

**Root cause** — Under `on-request` the harness deliberately does not auto-retry denials ("Under `Never` or `OnRequest`, do not retry without sandbox", `codex-rs/core/src/tools/orchestrator.rs:439-442`), and the reactive denial detector is keyword-based (`codex-rs/sandboxing/src/denial.rs:49-57`) — network failures under a no-network namespace produce DNS errors, not "permission denied"; network blocks are only semantic when they pass through the proxy (`Network access to "<host>" is blocked by policy.`).

**Fix · [[codex]]** — 2026-02-27 `6046ca19ba` prompt names DNS/host resolution, registry/index access and dependency download failures as escalation triggers (see [[sandbox-network-error-not-escalated]] for the prompt side). Proxy-routed denials carry a semantic `x-proxy-error` / approval retry reason ([[egress-policy-proxy]]); remote denials reported semantically (`9f06cf1a09`).

**Lesson** — When an isolation layer causes failures, surface them as typed "blocked by sandbox" results (or tell the model their signatures) — otherwise it retries or gives up instead of asking.

Related: [[tool-call-gate]] · [[tool-only-isolation]] · [[sandbox-escalation-retry]] · [[codex--sandbox-escalation-retry|codex]] · [[sandbox-network-error-not-escalated]]
