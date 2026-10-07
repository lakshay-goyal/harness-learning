---
type: failure
concepts: [plugin-tools]
harnesses: [pi, opencode]
---
**Symptom** — An extension registering a tool without a proper `parameters` object schema made provider requests fail (the whole tools array rejected), breaking the session, not just that tool (`packages/coding-agent/CHANGELOG.md:472`).

**Root cause** — `registerTool` accepted any value for `parameters`; validation happened only when the provider rejected the request.

**Fix · [[pi]]** — `acaa253cc` 2026-09-09: registration throws unless `parameters` is a non-array object ("must define an object parameter schema", `packages/coding-agent/src/core/extensions/loader.ts:289-295`).

**Fix · [[opencode]]** — `c79a9634d3` 2026-05-19 "tolerate plugin tool defs with missing args" (#28357): after a Zod upgrade, plugin tools without `args` crashed the registry (#27451, #27630); `args` now normalized to `{}` at the registry boundary (`packages/opencode/src/tool/registry.ts:125-135`). Tolerate-and-normalize, the opposite of pi's reject-at-registration.

**Lesson** — Validate plugin-supplied model-facing declarations at registration time; one bad plugin tool must not poison every request.

Related: [[plugin-tools]] · [[strict-tool-schema-rejections]] · [[pi--plugin-tools|pi]] · [[opencode--plugin-tools|opencode]]
