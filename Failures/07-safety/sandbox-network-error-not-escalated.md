---
type: failure
concepts: [permission-state-prompt]
harnesses: [codex]
---
**Symptom** — Under `-a on-request -s workspace-write`, "DNS/registry failures like `Could not resolve host: index.crates.io` could be treated like ordinary transient failures and not escalate" — the model retried or silently stopped instead of requesting network access (`6046ca19ba` body).

**Root cause** — Sandbox-induced network failures look identical to real outages; the prompt listed sandbox denials generically, not their concrete error signatures.

**Fix · [[codex]]** — 2026-02-27 `6046ca19ba` "Clarify escalation guidance for sandbox-related network failures (#13051)": "fails because of sandboxing or with a likely sandbox-related network error (for example DNS/host resolution, registry/index access, or dependency download failure), rerun the command with \"require_escalated\"" (`codex-rs/prompts/templates/permissions/approval_policy/on_request.md`). Same incident seen from the gate side: [[sandbox-failure-misread-as-transient]].

**Lesson** — The model cannot tell sandbox-induced failures from real ones; the prompt must enumerate the sandbox's failure signatures.

Related: [[permission-state-prompt]] · [[sandbox-escalation-retry]] · [[egress-policy-proxy]] · [[codex--permission-state-prompt|codex]]
