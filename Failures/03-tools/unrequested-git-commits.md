---
type: failure
concepts: [tool-description-design, shell-execution]
harnesses: [opencode]
---
**Symptom** — The model created git commits the user had not asked for.

**Root cause** — The shell description carried a Claude-Code-style commit procedure introduced with "When the user asks you to create a new git commit", which read as a standing workflow.

**Fix · [[opencode]]**
- `793542230f` 2025-11-19 "If and only if the user asks…".
- `d469d7d441` 2025-12-04 "IMPORTANT: ONLY COMMIT IF THE USER ASKS YOU TO."; `<commit_analysis>` wrapper removed.
- `548648a3d9` 2026-05-16 procedure collapsed; HEAD "Only commit, amend, push, or create PRs when explicitly requested." (`packages/opencode/src/tool/shell/shell.txt:14`).
- Prompts repeat it: "You are NEVER allowed to stage and commit files automatically." (`packages/opencode/src/session/prompt/beast.txt:147`); "NEVER commit changes unless the user explicitly asks you to." (`packages/opencode/src/session/prompt/default.txt:76`).

**Lesson** — Gate irreversible VCS actions on explicit request, stated in the tool that performs them.

Related: [[tool-description-design]] · [[shell-execution]] · [[amend-after-failed-commit]] · [[opencode--tool-description-design|opencode]]
