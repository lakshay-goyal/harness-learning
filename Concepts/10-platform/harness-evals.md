---
type: concept
stage: eval
tier: candidate
aliases: ["packages/evals", "pi-evals", "docs lift eval", "*.docs.eval.ts", "without_docs/with_docs", "documentation-audit eval", "createStorageConformance", "createEnvConformance", "createTelemetryAdapterConformance", "eval-harness-adapter", "documentation-lift-eval", "paired-arm-fail-closed-report", "counterbalanced-run-order", "eval-sandbox-privilege-drop", "state-oracle-grading", "doc-implementation-audit", "treatment-integrity-check", "scripted-faux-provider", "adapter-conformance-suite", "semantic-parity-differential-testing", "differential-reference-test", "validated-benchmark-workload"]
harnesses: [pi]
---
How a harness measures itself: an adapter wrapping the real agent for an eval framework, paired treatment/control arms, sandboxed runs, deterministic state-based grading, plus exported conformance and differential suites for pluggable backends.

## Why
- Model-facing changes (docs, prompts, tools) need evidence of lift, not anecdotes.
- Evals are easy to contaminate: control arm still sees the treatment ([[control-arm-docs-leakage]]), the agent can read judges ([[eval-judges-readable-by-agent]]), the harness tests a registry copy not the workspace ([[eval-harness-installs-registry-copy]]).
- Infrastructure failures masquerade as low scores unless separated ([[eval-assertions-conflated-with-scores]]); failed runs lose diagnostics ([[failed-run-diagnostics-lost]]).
- Treatment transforms silently stop applying when prompt structure changes ([[treatment-transform-marker-drift]], [[validation-reads-projected-not-sent-prompt]]).
- Pluggable backends (storage, exec env, telemetry adapters) drift from reference semantics without shared conformance suites ([[conformance-suite-platform-timing]], [[remote-errno-parity-drift]]).

## Design space
- **What is measured**: general coding benchmarks (SWE-bench, terminal-bench — pi uses none) vs harness-specific scenarios (pi: can the model customize pi from its docs; do docs match code).
- **Design**: single-arm pass rate vs paired A/B (pi `without_docs`/`with_docs`) with counterbalanced order (pi alternation) vs seeded shuffle (rejected, `743e1595c`).
- **Report policy**: average what you have vs fail-closed — withhold headline if any pair blocked; missing ≠ 0 (pi).
- **Grader**: LLM-as-judge vs deterministic (pi: all deterministic) vs state oracle — reload runtime from what the agent wrote and exercise it against a fixture server (pi).
- **Agent under test**: CLI subprocess vs in-process SDK session (pi, [[sdk-embedding]]).
- **Isolation**: temp dirs vs Docker per arm, read-only rootfs, privilege drop with read-probe of judges (pi).
- **CI**: model evals in CI vs only runner unit tests (pi).
- **Backend contracts**: exported runner-independent conformance suites (storage, env, telemetry) + random-sequence differential tests vs reference implementation + validated benchmark workloads (pi).

## Implementations
- [[pi--harness-evals|pi]] — `packages/evals` (vitest-evals) with Docker-isolated documentation-lift evals and host evals; deterministic judges; fail-closed paired report; pi-durable/pi-env/pi-telemetry conformance + differential suites.

## Failures
- [[eval-harness-installs-registry-copy]]
- [[eval-assertions-conflated-with-scores]]
- [[failed-run-diagnostics-lost]]
- [[bespoke-shuffle-order-bias]]
- [[treatment-transform-marker-drift]]
- [[validation-reads-projected-not-sent-prompt]]
- [[control-arm-docs-leakage]]
- [[eval-judges-readable-by-agent]]
- [[conformance-suite-platform-timing]]
- [[rewrite-drops-test-coverage]]

## Related
[[sdk-embedding]] · [[self-documentation-pointer]] · [[structured-tool-output]] · [[remote-execution-env]] · [[install-telemetry]] · [[durable-execution]] · [[spec-driven-agentic-development]]
