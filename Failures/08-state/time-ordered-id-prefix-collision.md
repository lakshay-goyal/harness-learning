---
type: failure
concepts: [session-tree]
harnesses: [pi]
---
**Symptom** — After entry ids switched to UUIDv7, short 8-char entry ids (prefix slice) collided constantly within a session and degenerated to full-UUID fallbacks (#6242).

**Root cause** — UUIDv7's leading bits are a millisecond timestamp; ids minted close together share the prefix, so `slice(0, 8)` has almost no entropy.

**Fix · [[pi]]** — `1dac09902` 2026-07-05 "derive short session entry ids from the uuidv7 random tail (closes #6242)": `slice(-8)` (`1dac09902:packages/agent/src/harness/session/jsonl-storage.ts:39`, historical: file deleted in `2132f8f75` 2026-07-31, whole agent-core harness removed in `7fd478a2e`). Coding-agent `SessionManager` uses `randomUUID()` (v4) `.slice(0, 8)` with ≤100 collision retries (`packages/coding-agent/src/core/session-manager.ts:276-284`) and was unaffected.

**Lesson** — Time-ordered ids have low-entropy prefixes; truncate from the random tail or use a random source for short ids.

Related: [[session-tree]] · [[pi--session-tree|pi]]
