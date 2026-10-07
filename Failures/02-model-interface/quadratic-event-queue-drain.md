---
type: failure
concepts: [unified-provider-api]
harnesses: [pi]
---
**Symptom** CPU time went quadratic draining large buffered `EventStream` queues: long streams or slow consumers stalled (#9055).

**Root cause** `Array.shift()` per event is O(n) on large arrays.

**Fix · [[pi]]** `b2602be77` 2026-09-07: two-stack FIFO queue (`packages/ai/src/utils/event-stream.ts:3-23`) (#9055).

**Lesson** Event-stream plumbing sits on the hot path of every token. Use O(1) queue operations.

Related: [[unified-provider-api]] · [[catalog-hot-path-quadratic]] · [[pi--unified-provider-api|pi]]
