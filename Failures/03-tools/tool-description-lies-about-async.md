---
type: failure
concepts: [tool-description-design, code-mode]
harnesses: [pi]
---
**Symptom** — Codemode scripts printed `{}` for discovery results: the model called `searchTools()` / `describeTool()` without `await` and serialized the pending promise; multiple outputs also ran together so the model could not tell them apart.

**Root cause** — The codemode tool description presented the discovery helpers as synchronous functions; the model wrote exactly what the "API doc" said.

**Fix · [[pi]]**
- `269121616` 2026-10-06 (#10555) — description marks lookup helpers async (`await searchTools(...)`) (`packages/coding-agent/src/extensions/codemode/tool.ts:147`).
- `eb326d265` 2026-10-06 — output items separated by `==> text N/M <==`, console lines grouped in `<console_output>` (`packages/coding-agent/src/extensions/codemode/execute.ts:256-281,528`).

**Lesson** — API signatures in a tool description are executable spec for the model: types and async-ness must be exact, and outputs must be unambiguously delimited.

Related: [[tool-description-design]] · [[code-mode]] · [[pi--code-mode|pi]] · [[pi--tool-description-design|pi descriptions]]
