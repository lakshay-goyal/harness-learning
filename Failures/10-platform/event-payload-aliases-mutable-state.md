---
type: failure
concepts: [agent-event-stream]
harnesses: [opencode]
---
**Symptom** — Clients saw token counts and part data change after the fact: a bus event carried the live part object, and later mutation by the processor altered values in events already published.

**Root cause** — Event payloads aliased mutable in-process state instead of carrying a snapshot.

**Fix · [[opencode]]** — `fd6f7133c5` 2026-03-02 "clone part data in Bus event to preserve token values" (#15780): `structuredClone` the payload at publish. Share sync clones diffs the same way (`packages/opencode/src/share/share-next.ts:198`).

**Lesson** — Publish immutable copies; an event is a snapshot of state at emit time.

Related: [[agent-event-stream]] · [[update-before-snapshot-on-subscribe]] · [[opencode--agent-event-stream|opencode]]
