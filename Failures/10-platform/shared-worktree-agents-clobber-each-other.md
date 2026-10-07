---
type: failure
concepts: [spec-driven-agentic-development, git-worktree-isolation]
harnesses: [pi, opencode]
---
**Symptom** — Several agent sessions working in one checkout (same cwd, different files) committed or destroyed each other's work: `git add -A`/`git add .` swept in another session's half-done files; `git stash`, `git checkout .`, `git reset --hard`, `git clean -fd` wiped other sessions' unstaged edits; branch switches for PR review moved the worktree under everyone.

**Root cause** — The harness gives no worktree isolation per session (no checkpoints, [[no-checkpoints-undo]]), and agents default to repo-wide git commands that assume a single writer.

**Fix · [[pi]]** — Policy, not mechanism. `5c59caee4` 2026-01-16 "docs: add critical git rules for parallel agent work" ("Multiple agents may work on different files in the same worktree"): commit only files you changed this session, stage explicit paths, `git status` before commit, banned-command list, never force-push, abort rebase if conflict is in a file you didn't modify (`AGENTS.md:52-72`). Extended to PR review: inspect via `gh pr diff` / `git show <ref>:<path>`, never `gh pr checkout` / `git switch` (`AGENTS.md:78-82`, `.pi/prompts/pr.md:4`). The `/wr` wrap prompt repeats "Commit only files you changed in this session" (`.pi/prompts/wr.md:24`).

**Fix · [[opencode]]** — mechanism, not policy: the harness creates a git worktree per parallel stream under `<data>/worktree/<projectID>` on branch `opencode/<name>` (`packages/opencode/src/worktree/index.ts:183-225`; `0b4af95223` 2026-01-03), usable as a local workspace target ([[git-worktree-isolation]]).

**Lesson** — When parallel agents share a worktree, every git verb that touches state outside the agent's own diff must be forbidden in the agent's standing instructions (or the harness must isolate each agent in its own worktree).

Related: [[spec-driven-agentic-development]] · [[context-file-hierarchy]] · [[pi]] · [[opencode--git-worktree-isolation|opencode]]
