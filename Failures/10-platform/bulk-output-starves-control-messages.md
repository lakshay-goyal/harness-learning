---
type: failure
concepts: [remote-execution-env, client-server-session-split]
harnesses: [pi]
---
**Symptom** — (design flaw, early pi-env) Over slow links, large command output and file payloads queued ahead of pings and cancel replies — the remote looked dead, cancels arrived late — and watching re-listed whole trees over SSH via a client-side polling watcher.

**Root cause** — Single FIFO writer; caller-side output windowing applied only after transfer; watcher ran far from the files.

**Fix · [[pi]]** — `46d0ff936` 2026-10-05 (+3154/−586): single stdout writer with control queue drained before bulk; replies with payload > 64 KiB to bulk (`packages/env/daemon/src/output.rs:1-99`; `main.rs:40-41,347-360`); unwindowed bulk blocks past 4 MiB unsent (`output.rs:13-14`); caller's tail window applied at source with exact `skipped` counts (`daemon/src/window.rs`; `protocol.md:104-107`); native watcher ported into the daemon (`watch.rs`; client `polling-watch.ts` removed `cd60a5b99`). Same pattern in the Radius relay writer (pending cap 64 MiB, 1 MiB drain threshold, `radius-relay.ts:14-15,496-536`).

**Lesson** — On constrained links, prioritize control over bulk, bound unsent bulk, drop what the consumer will discard at the source, and run watchers next to the data.

Related: [[remote-execution-env]] · [[client-server-session-split]] · [[shell-execution]] · [[pi--remote-execution-env|pi]]
