---
type: failure
concepts: [llm-approval-reviewer]
harnesses: [codex]
---
**Symptom** — Guardian approvals were based on evidence that changed mid-review: parent compaction removed evidence, or a new user message revoked permission while the review ran — and the stale *allow* was honoured. After the first fix, the opposite: any new user input (even "status?") aborted the pending action.

**Root cause** — An LLM verdict is a function of a transcript snapshot, but it was neither bound to that snapshot's revision nor re-evaluated selectively.

**Fix · [[codex]]** — 2026-09-07 `5b85aea979` "Keep Guardian review evidence consistent and reject stale approvals"; 2026-09-24 `49862b62be` "Retry Guardian reviews when authorization changes" — stale authorization is retryable within the attempt budget (`MAX_REVIEW_ATTEMPTS = 3`, `codex-rs/ext/guardian-reviewer/src/lib.rs:40`) because "the input was only a status inquiry"; deny if exhausted (`StaleAuthorization` → `FailedClosed`, `codex-rs/ext/guardian-reviewer/src/completion.rs:127-151`). 2026-08-25 `4b81410a80` `request_user_input` answers count as authorization changes; 2026-10-07 `3421c660d0` stale reviews rolled back before reviewer history is reused.

**Lesson** — Bind an LLM approval to the revision of the evidence it saw; on change, re-review rather than blindly honouring or discarding it.

Related: [[llm-approval-reviewer]] · [[delegated-authorization-provenance]] · [[codex--llm-approval-reviewer|codex]] · [[approval-reviewer-overcautious]]
