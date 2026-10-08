---
type: failure
concepts: [model-resolution]
harnesses: [pi]
---
**Symptom**
- `--model X` picked the first catalog provider (unauthenticated) when several providers shared the id.
- `zai/glm-5` matched a vercel-ai-gateway model whose id contains a slash.
- `xiaomi/mimo-v2.5-pro` failed when the inferred provider was unauthenticated but another authenticated provider had that raw id.
- OpenRouter ids containing colons (`x:exacto`) broke `:thinking` parsing.
- Bracketed ids in scoped patterns were treated as globs.
- Custom model ids lost their `:thinking` suffix or reasoning flag in the fallback path.
- Provider-scoped custom ids were rejected.

**Root cause** — A namespaced id grammar (`provider/id:level`) collides with gateway ids that contain `/` and `:`. Tie-breaks followed catalog order instead of usability.

**Fix · [[pi]]**
- `9a7863fc9` 2025-12-19 — try the full pattern first, then split on the **last** colon (#242) (`packages/coding-agent/src/core/model-resolver.ts:204-257`).
- `7364696ae` 2026-02-22 — prefer the provider/model split over gateway id matching (`:444-463`).
- `6f4bd814b` 2026-03-04 — provider-scoped custom model ids with a "Using custom model id" warning (#1759) (`:570-596`).
- `aa0398216` 2026-06-10 — parse the `:thinking` suffix in the fallback path.
- `1b2c32c65` 2026-06-12 — fall back to the authenticated raw id (#5643) (`:519-540`).
- `1fc80f4f6` 2026-06-12 — force `reasoning:true` when thinking is requested (`:589-591`).
- `da8dd8726` 2026-07-23 — exact match first, so bracketed ids are literals (`:306-312`).
- `b04faa2da` 2026-08-03 — if several providers match, choose the sole authenticated one; otherwise an ambiguity error with candidates and an auth hint (#7327/#7366) (`:470-504`).
- Strict CLI mode fails on an invalid suffix "to avoid accidentally resolving to a different model" (`:238-244`).

**Lesson** — Suffix and namespace syntax must try a literal match first. Ambiguous references should prefer the usable (authenticated) option or fail loudly, never fall back to catalog order.

Related: [[model-resolution]] · [[pi--model-resolution|pi]] · [[unusable-default-model-selected]]
