---
type: failure
concepts: [subagent-as-subprocess]
harnesses: [pi]
---
**Symptom** — Parallel subagent mode returned 100-char previews to the parent model, losing results and failure diagnostics (a child that exited before producing output gave the parent nothing actionable).

**Root cause** — The model-facing tool content was sized for the UI summary, not for the reader model; full results lived only in tool `details`.

**Fix · [[pi]]** — `93ecdbea3` 2026-05-18 "improve subagent parallel summaries" (#4710): each completed task's full final output, capped at `PER_TASK_OUTPUT_CAP` 50 KiB with an explicit `[Output truncated: N bytes omitted. Full output preserved in tool details.]` notice; failed tasks return errorMessage || stderr || final text (`packages/coding-agent/examples/extensions/subagent/index.ts:36,186-202,669-681`; `packages/coding-agent/examples/extensions/subagent/README.md:116-117,175`).

**Lesson** — The parent model only knows what the handoff string says; size it for the model, not the UI, and include failure diagnostics.

Related: [[subagent-as-subprocess]] · [[tool-output-truncation]] · [[structured-tool-output]] · [[pi--subagent-as-subprocess|pi]]
