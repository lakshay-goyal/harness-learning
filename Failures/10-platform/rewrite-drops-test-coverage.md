---
type: failure
concepts: [client-server-session-split, harness-evals]
harnesses: [pi]
---
**Symptom** — A large protocol rewrite silently removed client/server protocol tests; regressions went unguarded.

**Root cause** — `e52de91d0` 2026-08-13 "service-addressed session RPC" deleted 6,277 lines including the tests of the replaced surface.

**Fix · [[pi]]** — `ce142b2a6` 2026-08-14 "restore client-server protocol test coverage" (next day). Later the durable stack moved invariants into exported, runner-independent conformance suites (storage/env/telemetry) so rewrites must keep passing them (`packages/durable/src/testing/*`; `b45597504`).

**Lesson** — Rewrites should keep behavior tests as an external contract (conformance suite) rather than deleting them with the code they tested.

Related: [[client-server-session-split]] · [[harness-evals]] · [[spec-driven-agentic-development]] · [[pi--harness-evals|pi]]
