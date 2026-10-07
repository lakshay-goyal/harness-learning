---
type: concept
stage: permissions
tier: candidate
aliases: [guardian, auto_review, auto-review, approvals_reviewer, guardian_subagent, guardian_approval, strict_auto_review, codex-guardian-reviewer, codex-guardian-context, codex-guardian-v2, GuardianAssessment, GuardianRejectionCircuitBreaker, async risk scorer, Luna classifier, policy_template, tenant_policy_config, approval judge]
harnesses: [codex]
---
Route an approval request that would go to the human to a separate, sandboxed LLM reviewer that sees bounded transcript evidence (with host-assigned authorship provenance) plus the planned action and returns a structured allow/deny + risk + rationale; it only replaces the human at existing approval points and fails closed on any review error.

## Why
- Approval prompts either interrupt the user constantly or get rubber-stamped; unattended runs have nobody to ask.
- A deterministic gate can't judge intent ("did the user ask to delete this?", "is this payload exfiltration?"); an LLM can — if it is told which evidence is trusted ([[approval-reviewer-trusts-untrusted-content]], [[no-prompt-injection-defense]]).
- LLM judges drift: over-denial ([[approval-reviewer-overcautious]]), miscalibrated numeric scores ([[numeric-risk-score-miscalibrated]]), stale evidence ([[approval-invalidated-by-new-user-input]]), and regressions in a long policy prompt ([[prompt-edit-silently-reverted]]).
- After a denial the agent tends to grind on workarounds — needs a circuit breaker and explicit "do not circumvent" feedback ([[approval-circumvention]]).

## Design space
- **Human only** — pi has no approvals at all ([[no-permission-prompts]]); a pi extension could call a model inside [[tool-call-gate]] (none shipped).
- **Replace the human, keep the approval points** (✔ codex auto-review) vs **add review points** to actions config would skip (✔ codex `strict_auto_review`).
- **Reviewer runtime**: full subagent turn with read-only sandbox, restricted tools, `approval_policy=never` (✔ codex) vs single completion.
- **Output**: numeric risk score + threshold (codex MVP, removed) vs categorical `risk_level` + `user_authorization` → derived `outcome` (✔ codex); bare `{"outcome":"allow"}` for the cheap common case.
- **Evidence**: transcript as provenance-labelled JSON records with token budgets; trusted sources allow-listed (user/developer messages, AGENTS.md, `request_user_input` answers, host-verified skill paths) ✔ codex.
- **Failure semantics**: fail closed (✔ codex) except input-budget overflow when review is optional → ask the human.
- **Asynchronous pre-scoring**: cheap classifier scores actions in the background; a fresh low score short-circuits the blocking review (✔ codex Guardian V2, single-token `high`/`low`).
- **Loop control**: per-turn denial circuit breaker (✔ codex 3 consecutive / 10 of last 50; cyber 1/1).
- **Policy text ownership**: repo template + tenant/workspace policy + extra policy slot + model-catalog override (✔ codex).
- **Delegated agents**: judge a sub-agent's action against the *root* user's words ([[delegated-authorization-provenance]]).

## Implementations
- [[codex--llm-approval-reviewer|codex]] — Guardian / auto-review: hooks → guardian → user; read-only reviewer subagent, 90 s / 3 attempts, fail-closed; enum verdict; provenance-labelled evidence; circuit breaker; Guardian V2 async Luna scorer.

## Failures
- [[approval-reviewer-overcautious]]
- [[approval-reviewer-trusts-untrusted-content]]
- [[numeric-risk-score-miscalibrated]]
- [[prompt-edit-silently-reverted]]
- [[approval-invalidated-by-new-user-input]]

## Related
[[approval-policy-modes]] · [[delegated-authorization-provenance]] · [[sandbox-escalation-retry]] · [[structured-classifier-api]] · [[review-subagent]] · [[side-model-metadata-generation]] · [[tool-call-gate]] · [[tool-safety-annotations]] · [[xml-prompt-boundaries]] · [[no-prompt-injection-defense]] · [[isolation-strategy]]
