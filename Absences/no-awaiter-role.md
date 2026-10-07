---
type: absence
harnesses: [codex]
---
# no-awaiter-role

The built-in `awaiter` agent role is temporarily removed (commented out).

**What's missing**
- "Awaiter is temp removed" (`codex-rs/core/src/agent/role.rs:386`).

**Evidence of decision**
- `fe439afb81` 2026-02-27 "chore: tmp remove awaiter", after two days of description tuning (`81ce645733`, `c53c08f8f9` "calm down awaiter").

**Implication**
- Waiting on children is a generic mailbox wait tool, not a dedicated role ([[subagent-result-mailbox]]); status "temporary" per commit subject (unverified whether it returns).

Related: [[agent-profiles]] · [[subagent-result-mailbox]] · [[model-polls-background-work]] · [[Absences]]
