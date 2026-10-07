---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — The agent under test could read evaluator modules (judges, expected outputs) through Vite SSR transform caches in `/tmp`.

**Root cause** — Same user, shared tmp; transformed eval modules are plaintext files.

**Fix · [[pi]]** — `d7296c063` 2026-09-15: evaluator sources `chmod go-rwx` root-only (`docker/Dockerfile:27-35`); `enterToolSandbox` starts as root, chmods SSR caches 0600, `setgroups([])`/`setgid`/`setuid(65532)`, then verifies it CANNOT read protected modules (`packages/evals/src/harness.ts:130-187`); entrypoint probes from a UID-65532 child (`docker/entrypoint.ts:80-130`); credential file deleted and env var removed after auth resolution (`harness.ts:328-335`).

**Lesson** — An eval's expected answers and credentials are secrets from the model; drop privileges and prove unreadability.

Related: [[harness-evals]] · [[control-arm-docs-leakage]] · [[pi--harness-evals|pi]]
