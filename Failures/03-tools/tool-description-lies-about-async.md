---
type: failure
concepts: [tool-description-design, code-mode, plan-checklist-tool]
harnesses: [pi, codex]
---
**Symptom** — Codemode scripts printed `{}` for discovery results: the model called `searchTools()` / `describeTool()` without `await` and serialized the pending promise; multiple outputs also ran together so the model could not tell them apart.

**Root cause** — The codemode tool description presented the discovery helpers as synchronous functions; the model wrote exactly what the "API doc" said.

**Fix · [[pi]]**
- `269121616` 2026-10-06 (#10555) — description marks lookup helpers async (`await searchTools(...)`) (`packages/coding-agent/src/extensions/codemode/tool.ts:147`).
- `eb326d265` 2026-10-06 — output items separated by `==> text N/M <==`, console lines grouped in `<console_output>` (`packages/coding-agent/src/extensions/codemode/execute.ts:256-281,528`).

**Fix · [[codex]]** (weak mapping — a design correction, no recorded model symptom)
- `update_plan`'s tool definition carried behavioural instructions ("Skip a plan when…", "Planning steps are … like tasks or TODOs"); `30ee24521b` 2025-08-13 "remove behavioral prompting from update_plan tool def (#2261)" moved them to the main prompt (files `30ee24521b:codex-rs/core/prompt.md`, `30ee24521b:codex-rs/core/src/plan_tool.rs`), leaving a declarative description (`codex-rs/core/src/tools/handlers/plan_spec.rs:44-47`).
- Code-mode `exec` description states async-ness and output semantics explicitly ("unawaited promises are silently discarded", helper signatures) (`codex-rs/code-mode-protocol/src/description.rs:23-47`).

**Lesson** — A tool description is executable spec: signatures, async-ness and output delimiting must be exact, and usage policy belongs in the versioned system prompt rather than the schema.

Related: [[tool-description-design]] · [[code-mode]] · [[pi--code-mode|pi]] · [[pi--tool-description-design|pi descriptions]] · [[plan-checklist-tool]] · [[codex--tool-description-design|codex]]
