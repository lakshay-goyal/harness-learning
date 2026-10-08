---
type: failure
concepts: [tool-output-spill, tool-error-as-result]
harnesses: [opencode]
---
**Symptom** (latent, from code) — In the v2 runtime, if writing the managed tool-output file fails (disk full, permissions), a tool whose side effect already happened (a write, a shell command) is reported to the model as "Tool execution failed: Failed to write tool output: …", inviting it to redo the mutation.

**Root cause** — Two v2 documents disagree. `CONTEXT.md:194`: retention failure "does not change a successful tool operation into a failed one. The Session records an explicitly lossy bounded output without a path". `specs/v2/tools.md:157`: "if complete retention fails, settlement fails operationally rather than publishing lossy success". Code follows tools.md: `ToolOutputStore.StorageError` (`packages/core/src/tool-output-store.ts:30-37`, `:129-136`) propagates out of `bound` (`packages/core/src/tool/registry.ts:75`), the settlement fiber fails, and the runner calls `failUnsettledTools("Tool execution failed: <message>")` (`packages/core/src/session/runner/llm.ts:320-324`).

**Fix · [[opencode]]** — none at `ecc4916b5a`; introduced with `a9094fd059` 2026-06-05 (bounded v2 tool output). Legacy differs: spill write errors in `Truncate.output` are not caught either, but the wrapper runs inside `Effect.orDie` (`packages/opencode/src/tool/tool.ts:145`), so the call dies as a defect (inference).

**Lesson** — A post-success bookkeeping failure must not be reported as an operation failure; degrade to a lossy-but-honest result ("output truncated, full copy unavailable") and alert the operator, and pick one authoritative spec.

Related: [[tool-output-spill]] · [[tool-error-as-result]] · [[opencode--tool-output-spill|opencode impl]] · [[bash-output-integrity]]
