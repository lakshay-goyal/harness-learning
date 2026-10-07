---
type: failure
concepts: [replicated-state]
harnesses: [pi]
---
**Symptom** — Passing a settled (revoked) Chord draft to `Promise.resolve` / returning it from an async function threw, because the runtime probes `.then` on any object and the revoked overlay proxy rejected all access.

**Root cause** — Drafts are revoked when the change callback settles (spec invariant: drafts revoked, strict JSON); the revocation didn't special-case the thenable probe.

**Fix · [[pi]]** — `6966636db` 2026-09-24 "allow promise probes on settled drafts": `then` returns `undefined` on settled overlays. Chord consumer facades do the same (`packages/chord/src/services/consumer.ts:62`) so facades aren't thenable.

**Lesson** — Any Proxy that may leak into async code must answer `then` safely (return `undefined`).

Related: [[replicated-state]] · [[held-draft-reference-misaddresses]] · [[pi--replicated-state|pi]]
