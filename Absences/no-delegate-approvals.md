---
type: absence
harnesses: [codex]
---
# no-delegate-approvals

Internal delegate sessions (review, one-shot children, memory consolidation) cannot ask the user; their approval policy is forced to `never`.

**What's missing**
- `codex-rs/core/src/codex_delegate.rs:64-73` (`Constrained::allow_only(AskForApproval::Never)` at `:73`); ReviewTask forces Never (`codex-rs/core/src/tasks/review.rs:99-135`).

**Evidence of decision**
- Risky actions of V2 workers go through the automatic Guardian reviewer instead ([[llm-approval-reviewer]]), judged on the root user's genuine messages ([[delegated-authorization-provenance]]).

**Implication**
- Unattended children never block on a human; containment is sandbox + reviewer; repeated denials end the child turn with `TooManyDenials` (`codex-rs/core/src/session_prefix.rs:14`).

Related: [[delegate-session-runner]] · [[llm-approval-reviewer]] · [[delegated-authorization-provenance]] · [[approval-policy-modes]] · [[Absences]]
