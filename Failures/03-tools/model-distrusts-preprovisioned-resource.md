---
type: failure
concepts: [tool-description-design]
harnesses: [opencode]
---
**Symptom** — Given a pre-approved temp directory `${tmp}` in the shell description, the model tried to create or verify it before use.

**Root cause** — "Use this directory" does not tell the model the directory exists; models defensively `mkdir`/`ls` unknown paths.

**Fix · [[opencode]]** — `2283979199` 2026-04-30 pre-approved tmp dir; same day `3615d8e226` "clarify that temp directory already exists": "This directory has already been created, already exists, and is pre-approved for external access" (`packages/opencode/src/tool/shell/shell.txt:7`).

**Lesson** — State existence and permission of provisioned resources explicitly, even redundantly.

Related: [[tool-description-design]] · [[workspace-boundary-check]] · [[opencode--tool-description-design|opencode]]
