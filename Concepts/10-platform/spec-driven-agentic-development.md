---
type: concept
stage: architecture
tier: candidate
aliases: ["pico-v5-handoff", "Pico5 spec", "Package N", "docs(durable): specify …"]
harnesses: [pi]
---
Building harness subsystems by writing a normative specification first, then handing an ordered list of packages to a coding agent that implements one package at a time, runs checks, and stops for human review before the next.

## Why
- Large redesigns (durable runtime, protocol) done ad hoc churn: pi's remote stack went through 9 wire-breaking protocol versions in a month and several prototype generations (pico, pico2, pico3, mini, micro) before the spec-first Pico5.
- Agents implementing many packages at once drift or redesign later layers; a stop-per-package gate keeps scope bounded.
- Invariants written as spec contracts become conformance suites ([[harness-evals]]).

## Design space
- **Ad hoc iteration** (pi pre-2026-09 remote stack) vs **spike → spec → ordered handoff** (pi Pico5).
- **Granularity**: one spec commit precedes each implementation package (pi `docs(durable): specify …` → `Package N`).
- **Review gate**: continuous vs mandatory stop after every package (pi handoff text).
- **Footguns**: guard in code vs list as contracts in spec (pi spec lists structural footguns rather than guarding them).

## Implementations
- [[pi--spec-driven-agentic-development|pi]] — `packages/durable/docs/spec.md` + `pico-v5-handoff.md`; pi-durable built 2026-08-16 → 2026-10-06; pi-env created→released in one day.

## Failures
- [[rewrite-drops-test-coverage]]
- [[shared-worktree-agents-clobber-each-other]]

## Related
[[durable-execution]] · [[client-server-session-split]] · [[remote-execution-env]] · [[harness-evals]] · [[replicated-state]]
