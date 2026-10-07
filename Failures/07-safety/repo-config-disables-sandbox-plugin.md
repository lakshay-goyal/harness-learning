---
type: failure
concepts: [tool-only-isolation, project-trust-gate]
harnesses: [pi]
---
**Symptom** — (Latent, code-inferred; no exploit or issue recorded.) A user who installs the `sandbox/` example globally to OS-sandbox bash still lets any repository disable or widen that sandbox: the extension merges `<cwd>/.pi/sandbox.json` over global config with project precedence, and `enabled:false` or wider `allowWrite`/network lists take effect.

**Root cause** — The plugin reads a project file that is not in core's trust-gated list (`TRUST_REQUIRING_PROJECT_CONFIG_RESOURCES`, `packages/coding-agent/src/core/trust-manager.ts:30-39`), and never checks `ctx.isProjectTrusted()` (`packages/coding-agent/examples/extensions/sandbox/index.ts:79-108`; no `trust` reference in the file). A repo containing only `.pi/sandbox.json` doesn't even trigger the trust prompt.

**Fix · [[pi]]** — none at HEAD `b30a6dd77` (open). Remedy pattern: only honor project-scoped security config when `ctx.isProjectTrusted()`, or allow it to narrow but never widen the global policy.

**Lesson** — Safety plugins that read repo-local config re-open the trust hole unless they consult the trust decision; project config should only ever tighten a safety policy.

Related: [[tool-only-isolation]] · [[project-trust-gate]] · [[tool-call-gate]] · [[pi--tool-only-isolation|pi]] · [[no-sandbox]]
