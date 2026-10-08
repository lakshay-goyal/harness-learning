---
type: failure
concepts: [structured-compaction-summary]
harnesses: [pi, opencode]
---
**Symptom** — In multi-task sessions the compaction summary kept only one goal; other goals vanished after compaction.

**Root cause** — The first structured template (`ac71aac09`) constrained `## Goal` to "[1-2 sentences: …]".

**Fix · [[pi]]** — `a602e8aba` 2025-12-29 "Remove restrictive sentence limits from Goal section": now "[What is the user trying to accomplish? Can be multiple items if the session covers different tasks.]" (`packages/coding-agent/src/core/compaction/compaction.ts:511-512`).

**Fix · [[opencode]]** Two variants. (a) File paths placed "inside the section where they matter" vanished across compactions → `78f85b1cd6` 2026-07-07 (#35636) "ensure relevant files survive compaction": mandatory `## Relevant Files` section (`packages/core/src/session/compaction.ts:38-39`). (b) Small summarizers (DeepSeek V4 Flash) dropped prior-summary goals and workstreams on re-compaction → `dab2637217` 2026-08-12 (#42045): conversation before prior summary, "The <prior-summary> is discarded after this: anything you do not carry into the new summary is lost", "Carry forward objectives, constraints, user directives, decisions, and parallel workstreams … even when the <conversation> does not mention them" (`compaction.ts:47-50`).

**Lesson** — Template length constraints silently drop information; constrain format, not quantity, for fields that can be plural.

Related: [[structured-compaction-summary]] · [[iterative-summary-update]]
