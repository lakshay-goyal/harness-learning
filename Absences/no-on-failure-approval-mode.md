---
type: absence
harnesses: [codex]
---
# no-on-failure-approval-mode

The "ask only after a sandboxed failure" approval mode was removed; codex prefers model-initiated escalation with justification.

**What's missing**
- Survives only as a serde alias of on-request (`codex-rs/protocol/src/protocol.rs:1036`). Under OnRequest a denied command is returned to the model, not auto-retried (`codex-rs/core/src/tools/orchestrator.rs:439-441`).

**Evidence of decision**
- Deprecated 2026-02-12 `4668feb43a` ("performs worse than `on-request`", #11631); deleted 2026-06-23 `2cf2a6a844`.

**Implication**
- Escalation is a model decision ([[model-requested-permissions]], [[sandbox-escalation-retry]]) rather than a reactive retry prompt; old configs keep parsing ([[session-migration]]).

Related: [[approval-policy-modes]] · [[sandbox-escalation-retry]] · [[model-requested-permissions]] · [[no-permission-prompts]] · [[Absences]]
