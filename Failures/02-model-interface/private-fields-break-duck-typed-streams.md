---
type: failure
concepts: [unified-provider-api]
harnesses: [pi]
---
**Symptom** Adding response timing broke hand-rolled `EventStream` mocks in proxies and tests: they were no longer assignable to `StreamFn`.

**Root cause** True-private `#fields` on a widely-implemented class break structural typing for duck-typed implementers.

**Fix · [[pi]]** `3ba22ce17` 2026-10-06 moved the state to a WeakMap. `f284a2460` 2026-10-06 reverted to private fields and switched proxy and test mocks to the real `createAssistantMessageEventStream()` (`packages/ai/src/utils/event-stream.ts:132-135`). Timing is stamped with a monotonic `performance.now()` unless the message is already timed (`event-stream.ts:91-130`).

**Lesson** Adding true-private members to a class others imitate is a breaking change. Export a factory and require it.

Related: [[unified-provider-api]] · [[custom-provider-registration]] · [[pi--unified-provider-api|pi]]
