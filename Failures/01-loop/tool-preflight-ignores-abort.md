---
type: failure
concepts: [abort-propagation, parallel-tool-execution]
harnesses: [pi]
---
**Symptom** — After abort, tools kept going: an extension's `ctx.abort()` during one tool's confirmation dialog left sibling calls still preparing and prompting for confirmation, and queued Escape input was lost (#4276); prepared parallel tools still executed when abort happened during preflight (#8936).

**Root cause** — Two-phase (prepare → execute) pipeline checked the signal only once per batch: not after each `beforeToolCall` await (which can block on a human) and not at the prepare/execute boundary — prepared thunks ran unconditionally inside `Promise.all`.

**Fix · [[pi]]**
- `b94482762` 2026-05-19 — stop tool preflight after extension abort; discard `beforeToolCall` result if aborted during hook (`packages/agent/src/agent-loop.ts:738-744`, `757-760`; sequential `break` `:575-577`; parallel stop preparing `:613-615`).
- `afda4d620` 2026-09-01 — aborted guard in prepared thunks → "Operation aborted" without executing (`agent-loop.ts:619-628`, `641-643`) (#8936).

**Lesson** — Check cancellation after every await that can block (esp. on a human) and at every phase boundary of a prepare/execute pipeline.

Related: [[abort-propagation]] · [[parallel-tool-execution]] · [[tool-call-gate]] · [[pi--abort-propagation|pi]]
