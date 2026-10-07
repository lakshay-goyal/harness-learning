---
type: failure
concepts: [model-catalog, virtual-model-router]
harnesses: [pi]
---
**Symptom** — Model lookups slowed down with a large refreshed catalog. Prompt submission got slower as sessions grew longer.

**Root cause**
- The remote-overlay merge was O(n²).
- Branch model selection did a catalog lookup for every assistant message in the branch.

**Fix · [[pi]]**
- `c34f2d6ad` 2026-09-30 — merge keyed by `Map(type\0id)` (`packages/coding-agent/src/core/remote-catalog-provider.ts:31-35`).
- `a0660b174` 2026-09-30 — the last `model_change` wins only if virtual, otherwise the latest physical response wins; at most one catalog lookup (#10198) (`packages/coding-agent/src/core/virtual-models.ts:120-145`).

**Lesson** — Anything on the per-turn path must be O(1) or O(n) in catalog and session size.

Related: [[model-catalog]] · [[virtual-model-router]] · [[pi--virtual-model-router|pi]]
