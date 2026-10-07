---
type: concept
stage: permissions
tier: candidate
aliases: [user_authorization.rs, sender_context.rs, root_handoff.rs, root user authorization, sender provenance, TooManyDenials, GUARDIAN_NEXT_ACTION]
harnesses: [codex]
---
When an automatic approvals reviewer judges a sub-agent's risky action, the only authorization evidence is the *root* user's genuine messages (captured by the host at input admission, bounded, provenance-tagged); parent-authored delegation text, tool results, or forwarded "the user said OK" claims never count as consent.

## Why
- A sub-agent's transcript starts with a task written by another model; if the reviewer trusts it, any parent (possibly prompt-injected) can launder authorization ("user approved deleting prod") into a child.
- Conversely, without the root user's words the reviewer denies legitimate delegated work for "no user authorization" ([[approval-reviewer-overcautious]]).
- Compaction, handoffs, and resumed/unloaded threads drop or reorder evidence; provenance must survive them.

## Design space
- **No delegation-aware review** — pi (no core subagents, no reviewer).
- **Child judged on its own transcript** (parent's task text = user) — unsafe.
- **Root evidence projected into child review** (✔ codex): bounded root user messages, explicit omissions, assistant commentary excluded, sender messages captured only at turn-input admission.
- **Escalation on repeated denials**: child stops (`TooManyDenials`), parent told not to resume without explicit user confirmation (✔ codex).
- **Skill trust propagation**: invoked trusted root skills passed to workers (✔ codex).

## Implementations
- [[codex--delegated-authorization-provenance|codex]] — `user_authorization.rs` / `sender_context.rs` / `root_handoff.rs` project bounded root + sender evidence into Guardian reviews of workers; denial → `TooManyDenials` + `GUARDIAN_NEXT_ACTION`.

## Failures
- [[approval-reviewer-trusts-untrusted-content]]
- [[approval-invalidated-by-new-user-input]]

## Related
[[llm-approval-reviewer]] · [[in-process-subagent-threads]] · [[task-owned-subagent]] · [[subagent-config-inheritance]] · [[agent-message-board]] · [[no-prompt-injection-defense]]
