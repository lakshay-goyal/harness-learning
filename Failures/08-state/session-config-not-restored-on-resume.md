---
type: failure
concepts: [session-tree, thinking-level-abstraction, layered-settings]
harnesses: [pi]
---
**Symptom** — `--resume` reset thinking level to off (#342); resuming appended a spurious `thinking_level_change` entry (#1118); switching through a non-reasoning model persisted the capability-forced `off` as the saved default (#1864); `/model` and `/thinking` selections were persisted globally (#5263).

**Root cause** — Session-scoped config (model, thinking) was not recorded as session entries at creation, setters weren't idempotent, and *requested* level vs *effective* (clamped) level weren't separated; session choices leaked into global settings.

**Fix · [[pi]]**
- `98c85bf36` 2025-12-30 (#342) — save initial model and thinking level to the session (`packages/coding-agent/src/core/sdk.ts:448-459` at HEAD: new sessions append `model_change` + `thinking_level_change`; resume restores from entries `:210-277`).
- 0.50.8 (#1118) — idempotent `setThinkingLevel` (`packages/coding-agent/CHANGELOG.md:3807`).
- 0.56.3 (#1864) — keep saved default through non-reasoning models (`packages/coding-agent/CHANGELOG.md:3207`).
- 0.84.3 (#5263) — `/model`/`/thinking` saved globally only with Ctrl+S (`packages/coding-agent/CHANGELOG.md:642`).

**Lesson** — Record session configuration as log entries, keep user intent (requested) separate from capability clamps (effective), and don't let session-local choices write global settings implicitly.

Related: [[session-tree]] · [[thinking-level-abstraction]] · [[layered-settings]] · [[model-resolution]] · [[pi--session-tree|pi]]
