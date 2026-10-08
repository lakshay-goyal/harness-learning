---
type: failure
concepts: [model-resolution]
harnesses: [pi]
---
**Symptom**
- A saved default model whose provider was unauthenticated was selected at startup, blocking a working local custom model.
- Model and thinking changes leaked into global settings when they should have been session-only.

**Root cause** — Fallback steps in the startup cascade didn't check auth. Model selection persisted globally by default.

**Fix · [[pi]]**
- `ca09b2b1a` 2026-07-02 — the saved default is used **only if authenticated** (#6231) (`packages/coding-agent/src/core/model-resolver.ts:675-688`).
- Session restore (`restoreModelFromSession` `:714-783`) requires the model to exist AND be authenticated. Otherwise it falls back to the current model, then the default per provider, then the first available, with a `fallbackMessage`.
- `2ff8ba622` 2026-08-19 — model and thinking changes are session-scoped unless explicitly persisted (#8356) (`packages/coding-agent/src/core/agent-session.ts:2498-2510`).

**Lesson** — Every step of a fallback chain must check usability (auth), and silent global persistence of per-session choices surprises users.

Related: [[model-resolution]] · [[pi--model-resolution|pi]] · [[credential-resolution]] · [[model-reference-ambiguity]]
