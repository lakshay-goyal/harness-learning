---
type: failure
concepts: [shell-execution, tool-description-design]
harnesses: [opencode]
---
**Symptom** — When a commit failed (e.g. a pre-commit hook rejected it), the model ran `git commit --amend`, rewriting the previous, unrelated commit.

**Root cause** — Amend instructions assumed the commit had succeeded; the model did not check.

**Fix · [[opencode]]**
- `bb3509b5ff` 2026-04-24 "clarify git amend condition to require verification": amend only if the commit succeeded, verified with `git log`.
- `548648a3d9` 2026-05-16 simplified to "If a commit fails or hooks reject it, fix the issue and create a new commit; do not amend the failed commit." (`packages/opencode/src/tool/shell/shell.txt:18`).

**Lesson** — Destructive follow-ups must be conditioned on a verified prior state.

Related: [[shell-execution]] · [[unrequested-git-commits]] · [[opencode--tool-description-design|opencode]]
