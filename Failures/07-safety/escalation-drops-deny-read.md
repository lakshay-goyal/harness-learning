---
type: failure
concepts: [sandbox-escalation-retry, command-rule-policy]
harnesses: [codex]
---
**Symptom** — An admin `deny_read` rule (e.g. `**/*.env`) was enforced only by the sandbox, but approved escalation, execpolicy `allow` rules, and the known-safe-command list ran commands *unsandboxed* — so `cat .env` could read denied files after an unrelated approval.

**Root cause** — "Allowed to run" and "allowed to run without the sandbox" were the same decision; the sandbox was the only enforcer of deny-read, and every allow path removed it.

**Fix · [[codex]]** — 2026-05-11 `6506765168` "preserve managed deny-read during escalation"; 2026-05-28 `6e10142199` "preserve deny-read sandboxing for safe commands" ("`cat` or `ls` may be reasonable to allow … not valid for read-capable commands"). Now `unsandboxed_execution_allowed` = no denied reads (`codex-rs/core/src/tools/sandboxing.rs:290-294`), `require_escalated` downgraded to `UseDefault` when deny-reads exist (`codex-rs/core/src/tools/sandboxing.rs:296-310`), "a command allow rule does not authorize removing filesystem restrictions" (`codex-rs/core/src/tools/sandboxing.rs:255-263`); prompt: "Do not request escalation or additional permissions to read them; these denials are policy restrictions." (`codex-rs/prompts/src/permissions_instructions.rs:382-396`). Windows sandbox enforces managed deny-read too (`848cbad7f4`).

**Lesson** — An approval about *whether to run* must never be conflated with *running unsandboxed*; if the sandbox is the only enforcer of a rule, no approval path may remove it.

Related: [[sandbox-escalation-retry]] · [[command-rule-policy]] · [[model-requested-permissions]] · [[codex--sandbox-escalation-retry|codex]] · [[no-safe-command-allowlist]]
