---
type: failure
concepts: [install-telemetry]
harnesses: [pi]
---
**Symptom** — `packages/telemetry/README.md:365-383` still advertises `pi-agent-core` exports `AGENT_TELEMETRY_SCHEMAS`, `AI_TELEMETRY_SCHEMA`, `HARNESS_TELEMETRY_SCHEMA`, `startAiSpan`/`startHarnessSpan` that no longer exist; at HEAD no production code emits any span. Similarly the first-run analytics copy promises a `/privacy` command that does not exist (`first-time-setup.ts:77`).

**Root cause** — `7fd478a2e` 2026-10-01 deleted the old in-agent harness including telemetry schemas and pi-telemetry re-exports; the generated schema docs lived in another package.

**Fix · [[pi]]** — Not fixed at HEAD (b30a6dd77).

**Lesson** — Generated or cross-package docs disappear with their generator only if something checks them; tie docs to code with a reachability/audit check (pi's own documentation-audit eval covers only `packages/coding-agent/docs`).

Related: [[install-telemetry]] · [[harness-evals]] · [[pi--install-telemetry|pi]]
