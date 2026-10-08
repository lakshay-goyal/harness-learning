---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — The `without_docs` control arm could still read pi's docs: only the system-prompt docs section was stripped while README/docs/examples stayed installed and readable by tools (`git show d7296c063^:packages/evals/src/pi-harness.ts:54-62`). Measured lift was understated. (Causal link inferred from the diff; commit body empty — unverified.)

**Root cause** — Treatment defined as a prompt edit rather than a physical absence.

**Fix · [[pi]]** — `d7296c063` 2026-09-15 (#9635): per-variant Docker images; `without-docs-install` deletes coding-agent README/CHANGELOG/docs/examples (`packages/evals/docker/Dockerfile:14-19`); internal packages stripped symmetrically (`install-runtime.mjs:41-53`); entrypoint asserts `/repo` allowlist, no internal docs visible, variant-specific presence/absence (`docker/entrypoint.ts:27-62`).

**Lesson** — A treatment must be a physical, verified absence in the control arm.

Related: [[harness-evals]] · [[self-documentation-pointer]] · [[eval-judges-readable-by-agent]] · [[pi--harness-evals|pi]]
