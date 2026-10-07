---
type: failure
concepts: [review-subagent]
harnesses: [codex]
---
**Symptom** — `/review` sometimes made code changes instead of only reporting; a read-only review sandbox meant to stop it "cause[d] `/review` to fail in a variety of ways"; later the review delegate could re-enable collaboration (multi-agent) tools.

**Fix · [[codex]]**
- `291b54a762` 2025-12-04 read-only review sandbox → reverted `bbc5675974` 2025-12-16.
- `9f5b17de0d` 2026-02-18 "Disable collab tools during review delegation" — `Collab`, `MultiAgentV2` disabled "so the delegate cannot re-enable blocked tools", web search disabled, approval `Never` (`codex-rs/core/src/tasks/review.rs:105-121`); goals disabled (`codex-rs/core/src/session/review.rs:28-34`).
- Prompt side: "Do not generate a PR fix." (rubric); detached `review-agent` skill: "Do not modify files, create commits, push branches, post review comments, or delegate the review to another agent." (`83a4187837`).

**Lesson** — A reviewer sub-agent's restrictions must be enforced in its config, not only its prompt; sandbox-level restriction has collateral breakage.

Related: [[review-subagent]] · [[delegate-session-runner]] · [[tool-only-isolation]] · [[codex--review-subagent|codex]]
