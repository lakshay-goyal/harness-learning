---
type: failure
concepts: [xml-prompt-boundaries, context-file-hierarchy, system-prompt-override]
harnesses: [pi]
---
**Symptom** — Models inconsistently understood where injected AGENTS.md/CLAUDE.md content and system prompt sections began and ended; separately, custom system prompts glued the cwd line onto later appended content.

**Root cause** — Context files were injected under `# Project Context` / `## <path>` markdown headings, and the files themselves use `#`/`##` headings → nested/colliding structure. The custom-prompt path also lacked a newline after the cwd line.

**Fix · [[pi]]**
- `e2fd651eb` 2026-05-16 (merge `8e6913711`, #4541, external @herrnel): custom-prompt path → `<project_context>` + `<project_instructions path="…">` — "so that agents are less likely to ingest a prompt with inconsistent boundaries".
- `7577d3b8d` 2026-05-18 (merge `aad8cf660`, #4709): same for the default prompt ("…unclear boundaries").
- `3dd4623ee` 2026-08-10 (#7887): trailing newline after cwd — "custom system prompts concatenating the current working directory with later appended prompt content".
- `9e05370b2` 2026-09-16: every section XML-tagged (`system-prompt.ts:188-191`); HEAD render `system-prompt.ts:79-86`.
- `88619669e` 2026-05-06 (#4234): HTML export strips skill wrapper XML (UI side).

**Lesson** — Delimit injected documents with unambiguous tags (with a `path` attribute), never with headings that collide with the documents' own structure.

Related: [[xml-prompt-boundaries]] · [[context-file-hierarchy]] · [[system-prompt-override]] · [[pi--xml-prompt-boundaries|pi]]
