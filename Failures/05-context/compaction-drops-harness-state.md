---
type: failure
concepts: [auto-compaction, structured-compaction-summary, context-projection]
harnesses: [codex]
---
**Symptom** — Compaction replaces the model history window, which silently deleted harness-relevant facts that lived only in history items: MCP widget/resource origins needed to authorize later reads, delegated sub-agent tasks, host-verified `request_user_input` answers, Guardian approval evidence/authorizations, model/access-program pairs, turn attribution, reasoning effort, image-resize notices, multipart user text boundaries.

**Root cause** — The transcript doubles as the harness's state store; every feature that reads history implicitly assumes its items survive, but compaction keeps only user messages + a summary (or an opaque server item).

**Fix · [[codex]]** — one bounded, often model-invisible "retained context"/checkpoint per feature that survives compaction (~15 "Preserve X across compaction" commits Aug–Oct 2026):
- `a397079287` 2026-08-18 MCP resource origins (`codex-rs/codex-mcp/src/resource_origin.rs`).
- `4f6d06d485` 2026-07-30 delegated sub-agent tasks.
- `5971d42847` 2026-09-02 host-verified `request_user_input` answers.
- `0a12b855a0` 2026-08-30, `1c1e17782a` / `9f97cb79eb` 2026-08-31, `305eed102d` Guardian approval evidence/authorizations.
- `5f3180c793` 2026-09-25 model/access-program pairs; `35d9e4bc4d` 2026-09-08 reasoning effort.
- `551bd409eb` 2026-10-06 turn attribution.
- `4bd5b9fd09` 2026-08-04 image-resize notices; `bd3d4d1436` 2026-09-25 multipart user text boundaries.
- Checkpoint shape: `CompactedItem` carries `replacement_history`, retained/guardian context, window ids, latest token-usage record (`codex-rs/history/src/lib.rs:286-306`).

**Lesson** — When the transcript doubles as the state store, every feature that reads history needs an explicit compaction-survival contract (or its own store); otherwise compaction is a recurring silent-data-loss tax.

Related: [[auto-compaction]] · [[structured-compaction-summary]] · [[context-projection]] · [[file-op-tracking]] · [[codex--auto-compaction|codex]]
