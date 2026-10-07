---
type: failure
concepts: [permission-ruleset, tool-call-gate]
harnesses: [opencode]
---
**Symptom** — In the v2 runtime, when the user declined a permission request the tool returned an ordinary error; the model kept going and often attempted the same action again (or a workaround) instead of stopping.

**Root cause** — A human "no" was modeled as a tool failure, which models treat as something to route around.

**Fix · [[opencode]]** — `709af58612` 2026-07-04 "stop after declined permissions" (#35356): `PermissionV2.assert` converts `DeclinedError` into a defect (`packages/core/src/permission.ts:209`), the runner fails unsettled tools and interrupts the drain, ending the run; decline-with-feedback (`CorrectedError`) still continues. The legacy runtime already stopped on a plain reject (`ctx.blocked = ctx.shouldBreak`, `packages/opencode/src/session/processor.ts:200-202`) unless `experimental.continue_loop_on_deny` (`7368342bab` 2025-12-15).

**Lesson** — A user decline is a control signal that ends the turn, not a tool error the model may retry.

Related: [[permission-ruleset]] · [[tool-call-gate]] · [[tool-error-as-result]] · [[permission-feedback-lost-in-tool-error]] · [[opencode--permission-ruleset|opencode]]
