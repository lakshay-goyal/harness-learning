---
type: failure
concepts: [egress-policy-proxy]
harnesses: [codex]
---
**Symptom** — Several egress paths failed open: (a) DNS errors/timeouts in the private-IP pre-check returned "public" and allowed the request; (b) the default `NO_PROXY` bypassed `169.254.0.0/16`, incl. cloud metadata `169.254.169.254`, which clients evaluate before the proxy; (c) unapproved hostnames were resolved (leaking the name to DNS) and Seatbelt allowed direct port 53; (d) escalated (unsandboxed) exec still had managed-proxy state injected.

**Root cause** — Error branches and client-side bypass lists were treated as outside the policy; resolution happened before authorization.

**Fix · [[codex]]** — 2026-03-26 `aea82c63ea` "fail closed on network-proxy DNS lookup errors"; 2026-04-28 `3afb185a4f` "tighten network proxy bypass defaults"; 2026-10-07 `37eaae6eeb` "Require hostname authorization before proxy DNS lookups" (port-53 allowance removed); 2026-04-25 `9aaa5d9358` "Bypass managed network for escalated exec". Seatbelt fails closed when a proxy is configured but no loopback port is inferable (`codex-rs/sandboxing/src/seatbelt.rs:358-364`); deferred approvals fail waiters closed when the owner drops (`codex-rs/core/src/tools/network_approval.rs` header).

**Lesson** — Every error branch in an egress check must default to deny, authorization must precede resolution, and client-side bypass lists (`NO_PROXY`) are part of the policy surface.

Related: [[egress-policy-proxy]] · [[os-level-sandbox]] · [[codex--egress-policy-proxy|codex]] · [[sandbox-side-channel-syscalls]] · [[hook-error-fails-open]]
