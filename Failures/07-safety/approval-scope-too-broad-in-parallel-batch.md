---
type: failure
concepts: [tool-call-gate, parallel-tool-execution]
harnesses: [codex]
---
**Symptom** — With parallel tool calls, approving one exec request approved every request in the turn — approvals were keyed at turn level, not request level (`c4b771a16f` body).

**Root cause** — The approval key was coarser than the unit of action; parallelism turned a coarse key into a privilege escalation.

**Fix · [[codex]]** — 2026-02-10 `c4b771a16f` "update parallel tool call exec approval to approve on request id (#11162)". Same class earlier: 2025-07-23 `084236f717` call_id on patch approvals and elicitations; 2025-08-14 `6a0f709cff` call_id on MCP-server approval params. Session cache keys canonicalized per action (`codex-rs/core/src/tools/approvals.rs:161`, `:239-280`; [[approval-key-canonicalization]]).

**Lesson** — Approval decisions must be keyed by call id (and exact action); any coarser key becomes an escalation once calls run in parallel.

Related: [[tool-call-gate]] · [[parallel-tool-execution]] · [[approval-policy-modes]] · [[approval-key-canonicalization]] · [[codex--approval-key-canonicalization|codex]]
