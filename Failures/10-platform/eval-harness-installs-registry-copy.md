---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — The eval harness broke every pi release: npm installed pi's OWN packages from the registry under an alias, and version sync bumped those aliases to versions not yet published.

**Root cause** — Third-party `harness-pi-ai` had peer deps on the pre-rename `@mariozechner/pi-*` names; evals consumed a registry copy instead of the workspace code under test.

**Fix · [[pi]]** — `6173017a7` 2026-07-25: dropped `harness-pi-ai` and its legacy aliases; rebuilt on vitest-evals `createHarness`; `73c1696d9` shrank the adapter (~2100 lines → small). HEAD enforces: Docker images install packed workspace tarballs (`packages/evals/docker/install-runtime.mjs:16-26`), entrypoint asserts coding-agent resolves from `dist/index.js` (`docker/entrypoint.ts:102-105`).

**Lesson** — An eval harness must test the workspace code, never a registry copy of itself; assert the resolution.

Related: [[harness-evals]] · [[duplicate-host-module-instances]] · [[pi--harness-evals|pi]]
