---
type: failure
concepts: [tool-description-design, file-read-tool]
harnesses: [pi]
---
**Symptom** — The model read the first 2000 lines / 50KB of a file (or a pi doc), saw the truncation notice, and **stopped** — then acted on a partial view (inferred from the fix wording; issue text not fetched).

**Root cause** — The read description said "Use offset/limit for large files" but not that the model must keep reading when it needs the whole file; truncation was a soft hint, not a protocol.

**Fix · [[pi]]**
- `89636cfe6` 2026-01-24 — read description + "When you need the full file, continue with offset until complete." (`packages/coding-agent/src/core/tools/read.ts:96`).
- `d2de6d083` 2026-01-26 — system prompt `<docs>`: "Always read pi .md files completely and follow links to related docs".
- Notice format names the exact next call: `[Showing lines a-b of T. Use offset=N to continue.]` (`read.ts:184-199`).

**Lesson** — Tell the model the continuation protocol explicitly (when to continue and with which argument), in both the description and the truncation notice.

Related: [[tool-description-design]] · [[file-read-tool]] · [[tool-output-truncation]] · [[self-documentation-pointer]] · [[pi--file-read-tool|pi]]

**Fix · [[pi]]** (docs-pointer angle, appended by 04-prompting writer) — Earlier system-prompt attempts at the same behavior for pi's own docs: `d1465fa0c`, `57dc16d9b`, `84b663276`, `dbdb99c48`, `67a1b9581` 2025-12-31 — "When asked to create hooks, custom tools, themes, or skills: read the relevant docs AND examples, follow all .md cross-references" → per-topic map + "Always read the doc, examples, AND follow .md cross-references before implementing" (failure inferred: model built extensions without reading docs/examples). HEAD `<docs>` rules `system-prompt.ts:168-169`. See [[pi--self-documentation-pointer]].
