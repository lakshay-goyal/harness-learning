---
type: failure
concepts: [structured-compaction-summary]
harnesses: [pi]
---
**Symptom** — In multi-task sessions the compaction summary kept only one goal; other goals vanished after compaction.

**Root cause** — The first structured template (`ac71aac09`) constrained `## Goal` to "[1-2 sentences: …]".

**Fix · [[pi]]** — `a602e8aba` 2025-12-29 "Remove restrictive sentence limits from Goal section": now "[What is the user trying to accomplish? Can be multiple items if the session covers different tasks.]" (`packages/coding-agent/src/core/compaction/compaction.ts:511-512`).

**Lesson** — Template length constraints silently drop information; constrain format, not quantity, for fields that can be plural.

Related: [[structured-compaction-summary]] · [[iterative-summary-update]]
