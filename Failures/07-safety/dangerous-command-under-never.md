---
type: failure
concepts: [dangerous-command-heuristics, approval-policy-modes]
harnesses: [codex]
---
**Symptom** — With `approval_policy = never` (headless/CI), commands flagged dangerous (e.g. `rm -rf`) simply ran, because the "ask the user" branch had nobody to ask and fell through to allow.

**Root cause** — "Never ask" was implemented as "never block"; the dangerous-command check only ever produced a prompt.

**Fix · [[codex]]** — 2025-09-26 `55801700de` "reject dangerous commands for AskForApproval::Never (#4307)" — dangerous hit → `Forbidden` under Never (`codex-rs/core/src/exec_policy.rs:796-806`); rule `prompt` decisions likewise forbidden with `PROMPT_CONFLICT_REASON` "approval required by policy, but AskForApproval is set to Never" (`codex-rs/core/src/exec_policy.rs:48-49`); granular categories set to false auto-reject (`REJECT_*_REASON`, `:50-53`).

**Lesson** — "Never ask" must degrade to "deny the risky subset", not "allow everything".

Related: [[dangerous-command-heuristics]] · [[approval-policy-modes]] · [[command-rule-policy]] · [[codex--dangerous-command-heuristics|codex]] · [[codex--approval-policy-modes|codex]]
