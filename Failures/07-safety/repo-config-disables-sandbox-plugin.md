---
type: failure
concepts: [tool-only-isolation, project-trust-gate]
harnesses: [pi, opencode]
---
**Symptom** — (Latent, code-inferred; no exploit or issue recorded.) A user who installs the `sandbox/` example globally to OS-sandbox bash still lets any repository disable or widen that sandbox: the extension merges `<cwd>/.pi/sandbox.json` over global config with project precedence, and `enabled:false` or wider `allowWrite`/network lists take effect.

**Root cause** — The plugin reads a project file that is not in core's trust-gated list (`TRUST_REQUIRING_PROJECT_CONFIG_RESOURCES`, `packages/coding-agent/src/core/trust-manager.ts:30-39`), and never checks `ctx.isProjectTrusted()` (`packages/coding-agent/examples/extensions/sandbox/index.ts:79-108`; no `trust` reference in the file). A repo containing only `.pi/sandbox.json` doesn't even trigger the trust prompt.

**Fix · [[pi]]** — none at HEAD `b30a6dd77` (open). Remedy pattern: only honor project-scoped security config when `ctx.isProjectTrusted()`, or allow it to narrow but never widen the global policy.

**Fix · [[opencode]]** — same class addressed by design in the v2 runtime for provider-use policy: policy documents are read in reverse so user-global statements beat repository ones, org-managed policy is appended last and plugins cannot add statements — "a repository cannot silently re-enable something the user denied globally" (`specs/v2/provider-policy.md:154-202`, `specs/v2/provider-policy.md:162`; `9583e08be4` 2026-05-30). The legacy runtime's config layering still lets project config override global config, including `permission` (`packages/opencode/src/config/config.ts:371-565`).

**Lesson** — Safety plugins that read repo-local config re-open the trust hole unless they consult the trust decision; project config should only ever tighten a safety policy.

Related: [[tool-only-isolation]] · [[project-trust-gate]] · [[tool-call-gate]] · [[pi--tool-only-isolation|pi]] · [[no-sandbox]] · [[opencode--layered-settings|opencode]]
