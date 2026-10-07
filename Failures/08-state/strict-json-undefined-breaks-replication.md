---
type: failure
concepts: [replicated-state, client-server-session-split]
harnesses: [pi]
---
**Symptom** — Remote clients of the experimental session server never received the terminal transcript update for successful OpenAI Responses turns; the live view stayed stuck on the streaming state.

**Root cause** — The provider adapter wrote `output.errorMessage = undefined`. Chord/pi-protocol accept only strict JSON (`undefined`, non-finite numbers, cycles rejected — `packages/protocol/src/codec.ts:20-32`), so the replicated `Transcript` update containing an `undefined` property could not cross the protocol boundary, while in-process consumers never noticed.

**Fix · [[pi]]** — `1a7bc80e7` 2026-09-01 (with moving service wire semantics into Chord): `delete output.errorMessage` when absent — "Ensure successful OpenAI Responses messages remain strict JSON so terminal Transcript updates reach clients". Delta values placed into drafts are copied and validated as strict JSON (`packages/chord/src/delta/README.md:105-119`: `undefined` on an object key deletes, in arrays throws).

**Lesson** — With a strict-JSON wire, `undefined` ≠ absent is a correctness bug; validate at producers, not only at the boundary, and test the remote path, not just in-process.

Related: [[replicated-state]] · [[client-server-session-split]] · [[errors-as-stream-events]] · [[pi--replicated-state|pi]]
