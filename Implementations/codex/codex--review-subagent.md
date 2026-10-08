---
type: implementation
harness: codex
concept: review-subagent
commit: 622e9e3696
files: [codex-rs/core/src/tasks/review.rs:105, codex-rs/core/src/tasks/review.rs:119, codex-rs/core/src/tasks/review.rs:190, codex-rs/core/src/tasks/review.rs:227, codex-rs/prompts/templates/review/rubric.md, codex-rs/prompts/src/review_request.rs:9, codex-rs/prompts/templates/review/exit_success.xml, codex-rs/protocol/src/protocol.rs:3499, codex-rs/core/src/session/review.rs:28]
---
[[review-subagent]] in [[codex]].

## Mechanism
1. **Targets**: uncommitted changes (staged + unstaged + untracked), diff vs base branch, a commit, custom instructions (`ReviewTarget`, `codex-rs/protocol/src/protocol.rs:3499-3522`). Target prompts are Rust constants: `UNCOMMITTED_PROMPT` "Review the current code changes (staged, unstaged, and untracked files) and provide prioritized findings."; `BASE_BRANCH_PROMPT` with precomputed `{{merge_base_sha}}` ("Run `git diff {{merge_base_sha}}`") and a BACKUP variant telling the model to compute `git merge-base` itself; `COMMIT_PROMPT(_WITH_TITLE)` (`codex-rs/prompts/src/review_request.rs:18-32`).
2. **Child setup** (`codex-rs/core/src/tasks/review.rs:105-141`): base instructions replaced with `REVIEW_PROMPT` (`:119`, provenance Custom `:120`) = `codex-rs/prompts/templates/review/rubric.md` (7,685 bytes, embedded `codex-rs/prompts/src/review_request.rs:9`); approval policy constrained to `Never` (`:121`); web search disabled; `Collab` and `MultiAgentV2` disabled "so the delegate cannot re-enable blocked tools" (`:115-116`); model = `review_model` or current (`:124`). Session-level: web search request/cached and **Goals** disabled for reviews (`codex-rs/core/src/session/review.rs:28-34`). If the review model rejects the parent's effort, the middle supported level is chosen (`supported_reasoning_levels.get((len-1)/2)`, `codex-rs/core/src/session/review.rs`).
3. **Runner**: `run_codex_thread_one_shot` — auto-shutdown on first terminal event, closed submit channel (`codex-rs/core/src/codex_delegate.rs:215-322`) → [[delegate-session-runner]]; task kind Review in the single slot, not steerable, ends with `TurnAbortReason::ReviewEnded` ([[single-active-task-slot]]).
4. **Rubric** content: 8 "is it a bug" criteria (e.g. "The bug was introduced in the commit (pre-existing bugs should not be flagged)", "It is not enough to speculate that a change may disrupt another part of the codebase … one must identify the other parts of the code that are provably affected"); 8 comment rules ("should not claim that an issue is more severe than it actually is", "no chunks of code longer than 3 lines", avoid "Great job ...", "Thanks for ..."); "If there is no finding that a person would definitely love to see and fix, prefer outputting no findings. Do not stop at the first qualifying finding."; `[P0]`–`[P3]` title tags + numeric priority; "The code_location should overlap with the diff."; "Do not generate a PR fix."; "**Do not** wrap the JSON in markdown fences"; precedence clause: "These are not the final word … other, more specific guidelines … may be present elsewhere in a developer message, a user message, a file … Those guidelines should be considered to override these general instructions." (rubric lines 5-8); "## Repository Rule Attribution": apply `AGENTS.override.md` / `AGENTS.md` precedence, cite the instruction file's "smallest supporting line range", "Do not fabricate citations or add hidden metadata or output fields" → [[context-file-hierarchy]].
5. **Output**: reviewer's assistant streaming suppressed; final message parsed as `ReviewOutputEvent` JSON — findings[title ≤ 80 chars, body, confidence_score, priority, code_location{absolute_file_path, line_range}], overall_correctness, overall_explanation, overall_confidence_score — fallback to the first `{…}` substring, then plain text in `overall_explanation` (`codex-rs/core/src/tasks/review.rs:145-210`; `codex-rs/protocol/src/protocol.rs:3534-3566`) → [[structured-tool-output]].
6. **Return to parent**: rendered user message + assistant message in parent history (`codex-rs/core/src/tasks/review.rs:212-279`): `render_review_exit_success` (`:227`) → `<user_action><context>User initiated a review task. Here's the full review output from reviewer model. User may select one or more comments to resolve.</context><action>review</action><results>{{results}}</results></user_action>` (`codex-rs/prompts/templates/review/exit_success.xml`); interrupted (`:231`): "User initiated a review task, but was interrupted. If user asks about this, tell them to re-initiate a review with `/review` and wait for it to complete." (`codex-rs/prompts/templates/review/exit_interrupted.xml`) → [[side-task-result-invisible-to-parent]].
7. **Detached review**: app-server `review/start` with detached delivery forks a new thread via `AgentRunner` (refused for paginated-history threads) (`codex-rs/app-server/src/request_processors/turn_processor.rs:1464-1500`; `codex-rs/ext/agent/src/lib.rs:33-101`) with a target-specific prompt referencing the bundled skill `codex-rs/skills/src/assets/samples/review-agent/SKILL.md` ("Perform a read-only, defect-first review… Do not modify files, create commits, push branches, post review comments, or delegate the review to another agent."; base-branch reviews `git merge-base HEAD <comparison-ref>` against upstream) (`83a4187837`). Two review prompt stacks coexist: rubric-as-base-instructions (inline) and skill (detached) → [[skill-progressive-disclosure]].

## Constants
| name | value | path:line |
|---|---|---|
| rubric code chunk / range / title | ≤ 3 lines code; ranges ≤ 5–10 lines; title ≤ 80 chars | `codex-rs/prompts/templates/review/rubric.md` |
| priority tags | `[P0]`–`[P3]` | `codex-rs/prompts/templates/review/rubric.md` |
| review approval policy | `Never` | `codex-rs/core/src/tasks/review.rs:121` |

## Evolution
- 2025-09-12 `90a0fd342f` "Review Mode (Core) (#3401)" — rubric essentially unchanged since (list-spacing fix `f842849bec`).
- 2025-09-16 `72733e34c4` "Add dev message upon review out (#3758)" — parent history record.
- 2025-10-29 `13e1d0362d` "Delegate review to codex instance (#5572)" — in-thread mode → delegated child.
- 2025-11-28 `aaec8abf58` "feat: detached review".
- 2025-12-02 `4b78e2ab09` "chore: review everywhere (#7444)" — merge-base precomputation.
- 2025-12-04 `291b54a762` read-only review sandbox; reverted 2025-12-16 `bbc5675974` → [[review-agent-edits-code]].
- 2026-01-08 `be212db0c8` AGENTS.md + skills included in `/review` child; 2026-01-14 `92472e7baa` "Use current model for review".
- 2026-02-18 `9f5b17de0d` "Disable collab tools during review delegation".
- 2026-06-01 `ba2b67f9cd` "Consolidate shared prompts in codex-prompts (#25151)" — `exit_success.xml` / `exit_interrupted.xml` added under `codex-rs/prompts/templates/review/`.
- 2026-07-14 `83a4187837` bundled `review-agent` skill for detached reviews.
- 2026-07-21 `81de4f251c` "Attribute review findings to repository rules (#34637)".

## Quirks
- `codex-rs/core/templates/review/history_message_completed.md` / `history_message_interrupted.md` still exist but no Rust code references them (live text comes from `codex-rs/prompts/templates/review/exit_*.xml` via `render_review_exit_success`, `codex-rs/core/src/tasks/review.rs:3-4`) — orphaned copies (same pattern as the orphaned orchestrator/collab templates in [[codex--task-owned-subagent|task-owned-subagent]]).
- The rubric is one of the few Codex prompts stable for a year (unverified inference: tuned before open-sourcing).

## Versus pi
- pi example `reviewer` role is a subprocess agent with a role prompt appended ([[pi--subagent-as-subprocess]]); codex replaces the child's base instructions with the rubric, locks it down in config, parses JSON and re-injects a `<user_action>` record.
