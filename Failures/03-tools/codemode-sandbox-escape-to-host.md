---
type: failure
concepts: [code-mode]
harnesses: [pi]
---
**Symptom** — Model-written codemode scripts took down the **host** pi process rather than failing inside the sandbox: a script printing in a loop exhausted host memory (OOM crash); a script patching built-ins (e.g. `Array.prototype.toJSON`) crashed the host and left the tool call unsettled.

**Root cause** — The sandbox boundary covered syscalls (no fs/net/timers in QuickJS) but not **serialization and resource limits**: output accumulated unbounded on the host, and the host trusted VM-side JSON serialization of bridge messages.

**Fix · [[pi]]**
- `319fecb89` 2026-10-02 (#10283) — hard caps inside the VM: `MAX_OUTPUT_CHARS = 16Mi`, `MAX_OUTPUT_ITEMS = 100000` → RangeError (`packages/codemode/src/runtime/prelude-source.ts:40-41,344-346`).
- `b223082bb` 2026-10-05 (#10444) — built-ins frozen and globals read-only before the script runs; host validates every worker payload and maps malformed ones to a `BridgeError` ("The script may have modified built-ins such as a prototype's toJSON") instead of crashing (`packages/codemode/src/runtime/host.ts:49-95,200-213`).
- Pre-existing: `CODEMODE_MEMORY_LIMIT_BYTES = 256MB` (worker shares pi's process; wasm32 could otherwise grow to 4GiB) (`packages/coding-agent/src/extensions/codemode/execute.ts:51-56`); fresh worker + VM per execution killed with `terminate()` (`host.ts:120-124`).

**Lesson** — Treat sandbox→host messages as untrusted input and cap resources inside the sandbox; the boundary includes serialization and memory, not just APIs.

Related: [[code-mode]] · [[nested-tool-calls]] · [[no-sandbox]] · [[pi--code-mode|pi]]
