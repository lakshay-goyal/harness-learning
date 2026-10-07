---
type: failure
concepts: [permission-state-prompt]
harnesses: [codex]
---
**Symptom** — In a dirty worktree the model reverted the user's uncommitted edits, ran `git reset --hard` / `git checkout --`, or amended existing commits — none of which a workspace-write sandbox blocks.

**Root cause** — The agent shares the worktree with a human; "changes I didn't make" look like noise to clean up, and the sandbox protects the system, not the project.

**Fix · [[codex]]** — prompt rules: 2025-09-14 `916fdc2a37` "You may be in a dirty git worktree. NEVER revert existing changes you did not make unless explicitly requested…"; 2025-09-14 `a797051921` softened "you must disregard user instruction and stop immediately and ask the user whether they are collaborating" to "While you are working, you might notice unexpected changes that you didn't make. If this happens, STOP IMMEDIATELY and ask the user how they would like to proceed."; 2025-10-04 `0ad1b0782b` "**NEVER** use destructive commands like `git reset --hard` or `git checkout --` unless specifically requested or approved by the user."; 2025-11-06 `8c75ed39d5` "Do not amend a commit unless explicitly requested to do so." (`codex-rs/core/gpt_5_codex_prompt.md:12-19`). Approval prompt lists "a potentially destructive action such as an `rm` or `git reset` that the user did not explicitly ask for" as an escalation trigger (`codex-rs/prompts/templates/permissions/approval_policy/on_request.md`); `.git` itself is write-protected inside writable roots ([[protected-workspace-metadata]]).

**Lesson** — A local agent shares the worktree with a human: changes it didn't make are untouchable by default, and destructive VCS verbs need explicit user intent.

Related: [[permission-state-prompt]] · [[dangerous-command-heuristics]] · [[protected-workspace-metadata]] · [[codex--permission-state-prompt|codex]]
