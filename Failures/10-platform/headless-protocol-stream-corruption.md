---
type: failure
concepts: [headless-rpc-mode, agent-event-stream]
harnesses: [pi]
---
**Symptom** — Headless clients saw broken or hung protocol streams:
- payloads containing U+2028/U+2029 split one JSON record into two (#1911);
- stray `console.log`/package-manager chatter interleaved with JSON on stdout (#2388, #2482);
- unknown-command errors lacked the request `id`, so clients waited forever; listener unsubscribe skipped events; output lost under pipe backpressure (#9990, #5868, #4897).

**Root cause** — Node `readline` treats Unicode line separators as line breaks; stdout was shared between protocol and incidental writers; raw writes ignored ENOBUFS/EAGAIN.

**Fix · [[pi]]**
- `e3adaf1bd` 2026-03-07 — LF-only JSONL reader/serializer (`packages/coding-agent/src/modes/rpc/jsonl.ts:4-58`, `serializeJsonLine` `:10`).
- `21ef72e9c` 2026-03-20 — protect RPC stdout (`packages/coding-agent/src/modes/rpc/rpc-mode.ts:24,55` takeOverStdout); `f1fe49a64` 2026-03-22 — `takeOverStdout()` redirects all `process.stdout.write` to stderr in print/json/rpc (`src/main.ts:651-655`; `core/output-guard.ts:45-70`).
- `d0d1d8edc` 2026-05-24 — honor stdout backpressure with ENOBUFS/EAGAIN retry (`output-guard.ts:20-43`).
- Example `chalk-logger` removed for "breaks TUI by using console.log directly" (`c5c515f56`).

**Lesson** — A headless agent protocol needs its own framing (never a generic line reader) and exclusive ownership of stdout.

Related: [[headless-rpc-mode]] · [[agent-event-stream]] · [[unicode-sanitization]] · [[pi--headless-rpc-mode|pi]]
