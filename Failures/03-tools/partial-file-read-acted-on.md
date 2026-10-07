---
type: failure
concepts: [tool-description-design, file-read-tool, skill-progressive-disclosure]
harnesses: [pi, codex]
---
**Symptom** — The model read the first 2000 lines / 50KB of a file (or a pi doc), saw the truncation notice, and **stopped** — then acted on a partial view (inferred from the fix wording; issue text not fetched).

**Root cause** — The read description said "Use offset/limit for large files" but not that the model must keep reading when it needs the whole file; truncation was a soft hint, not a protocol.

**Fix · [[pi]]**
- `89636cfe6` 2026-01-24 — read description + "When you need the full file, continue with offset until complete." (`packages/coding-agent/src/core/tools/read.ts:96`).
- `d2de6d083` 2026-01-26 — system prompt `<docs>`: "Always read pi .md files completely and follow links to related docs".
- Notice format names the exact next call: `[Showing lines a-b of T. Use offset=N to continue.]` (`read.ts:184-199`).

**Fix · [[codex]]** (skills variant)
- Symptom: the skills prompt said "open its `SKILL.md`. Read only enough to follow the workflow." and "load only the specific files needed…", so the model acted on truncated/paginated SKILL.md reads and handed skill interpretation to subagents, who summarized it lossily.
- `56554904ba` 2026-06-08 "Require complete main-agent skill reads (#27044)" — "read its `SKILL.md` completely before taking task actions. If a read is truncated or paginated, continue until EOF.", "Do not delegate reading, summarizing, or interpreting skill instructions to a subagent.", "Progressive disclosure applies to selecting relevant files, not partially reading a selected instruction file." (`codex-rs/ext/skills/src/catalog_prompt.rs:28-30`).
- codex has no read tool, so the continuation protocol lives in the prompt, not in a tool notice ([[file-read-tool]]).

**Lesson** — Tell the model the continuation protocol explicitly (when to continue and how) wherever reads can be cut — description, notice or prompt — and let progressive disclosure choose which files to read, never how much of a chosen instruction file.

Related: [[tool-description-design]] · [[file-read-tool]] · [[tool-output-truncation]] · [[self-documentation-pointer]] · [[pi--file-read-tool|pi]] · [[skill-progressive-disclosure]] · [[codex--file-read-tool|codex]]

**Fix · [[pi]]** (docs-pointer angle, appended by 04-prompting writer) — Earlier system-prompt attempts at the same behavior for pi's own docs: `d1465fa0c`, `57dc16d9b`, `84b663276`, `dbdb99c48`, `67a1b9581` 2025-12-31 — "When asked to create hooks, custom tools, themes, or skills: read the relevant docs AND examples, follow all .md cross-references" → per-topic map + "Always read the doc, examples, AND follow .md cross-references before implementing" (failure inferred: model built extensions without reading docs/examples). HEAD `<docs>` rules `system-prompt.ts:168-169`. See [[pi--self-documentation-pointer]].
