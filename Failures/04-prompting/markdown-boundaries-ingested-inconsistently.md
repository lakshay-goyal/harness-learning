---
type: failure
concepts: [xml-prompt-boundaries, context-file-hierarchy, system-prompt-override, message-role-layering]
harnesses: [pi, codex]
---
**Symptom** — Models inconsistently understood where injected AGENTS.md/CLAUDE.md content and system prompt sections began and ended; separately, custom system prompts glued the cwd line onto later appended content.

**Root cause** — Context files were injected under `# Project Context` / `## <path>` markdown headings, and the files themselves use `#`/`##` headings → nested/colliding structure. The custom-prompt path also lacked a newline after the cwd line.

**Fix · [[pi]]**
- `e2fd651eb` 2026-05-16 (merge `8e6913711`, #4541, external @herrnel): custom-prompt path → `<project_context>` + `<project_instructions path="…">` — "so that agents are less likely to ingest a prompt with inconsistent boundaries".
- `7577d3b8d` 2026-05-18 (merge `aad8cf660`, #4709): same for the default prompt ("…unclear boundaries").
- `3dd4623ee` 2026-08-10 (#7887): trailing newline after cwd — "custom system prompts concatenating the current working directory with later appended prompt content".
- `9e05370b2` 2026-09-16: every section XML-tagged (`system-prompt.ts:188-191`); HEAD render `system-prompt.ts:79-86`.
- `88619669e` 2026-05-06 (#4234): HTML export strips skill wrapper XML (UI side).

**Fix · [[codex]]**
- Symptom variant: after AGENTS.md started being injected as a user message (PR #1737), "the model confus[ed] AGENTS.md context as part of the message" (`063083af15` body).
- `063083af15` 2025-08-04 "[prompts] Better user_instructions handling (#1836)": wrapped in `<user_instructions>\n\n…\n\n</user_instructions>` (USER_INSTRUCTIONS_START/END in `codex-rs/core/src/client_common.rs`) + prompt lines "`user_instructions` are not part of the user's request, but guidance for how to complete the task." / "Do not cite `user_instructions` back to the user unless a specific piece is relevant." (removed again in `81b148bda2`).
- `2371d771cc` 2025-10-30: format `# AGENTS.md instructions for <dir>` + `<INSTRUCTIONS>` (now `codex-rs/core/src/context/user_instructions.rs`); every injected fragment gets open/close markers (`codex-rs/context-fragments/src/fragment.rs`).

**Lesson** — Delimit injected documents with unambiguous tags (with a `path` or directory label), never with headings that collide with the documents' own structure; when they ride in a user-role message, also say they are guidance, not the request.

Related: [[xml-prompt-boundaries]] · [[context-file-hierarchy]] · [[system-prompt-override]] · [[pi--xml-prompt-boundaries|pi]] · [[message-role-layering]] · [[codex--xml-prompt-boundaries|codex]]
