---
type: concept
stage: subagents
tier: candidate
aliases: [review-subagent-rubric, "/review", ReviewTask, "Op::Review", ReviewRequest, ReviewTarget, ReviewOutputEvent, ReviewFinding, REVIEW_PROMPT, review_model, "SubAgentSource::Review", "<user_action>", detached review, review-agent skill]
harnesses: [codex]
---
Code review run as a one-shot, locked-down child session whose base instructions are wholly replaced by a review rubric with a strict JSON findings schema; the verdict is parsed and folded back into the parent transcript as a tagged record so the main agent can act on it.

## Why
- A reviewer that shares the main agent's prompt, tools and context is biased toward its own changes and may "fix" instead of review ([[review-agent-edits-code]]).
- Without a model-visible record the main agent can't act on findings the user just saw ([[side-task-result-invisible-to-parent]]).
- A fixed rubric + schema makes findings rankable (priority, confidence) and renderable.

## Design space
- **Host**: same thread in a "review mode" (codex 2025-09) · delegated one-shot child (✔ codex inline review) · forked detached thread driven by a bundled skill (✔ codex detached review) · subprocess role (pi example `reviewer`, [[subagent-as-subprocess]]).
- **Prompt**: role prompt appended · rubric replaces base instructions (✔ codex `REVIEW_PROMPT`).
- **Lockdown**: prompt-only · config-enforced: approval `never`, web search off, multi-agent tools off, goals off (✔ codex) · read-only sandbox (codex tried, reverted).
- **Output**: free text · JSON schema with findings[title, body, confidence_score, priority, code_location] + overall verdict; fallback parsing (✔ codex).
- **Return path**: UI only · `<user_action>` user message + assistant message in parent history; interrupted variant tells user to re-run (✔ codex).
- **Targets**: uncommitted / base branch (merge-base precomputed) / commit / custom (✔ codex).

## Implementations
- [[codex--review-subagent|codex]] — `ReviewTask` via `run_codex_thread_one_shot`; `codex-rs/prompts/templates/review/rubric.md`; JSON `ReviewOutputEvent`; `exit_success.xml` reinjection; detached review via `AgentRunner` + `review-agent` skill.

## Failures
- [[review-agent-edits-code]]
- [[side-task-result-invisible-to-parent]]

## Related
[[delegate-session-runner]] · [[single-active-task-slot]] · [[in-process-subagent-threads]] · [[subagent-as-subprocess]] · [[structured-tool-output]] · [[context-file-hierarchy]] · [[skill-progressive-disclosure]] · [[out-of-band-message-deferral]] · [[subagent-hosting]]
