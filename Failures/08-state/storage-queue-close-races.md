---
type: failure
concepts: [durable-execution]
harnesses: [pi]
---
**Symptom** — pi-durable's async SQLite adapter raced between its call queue, transactions and `close()`: work could start before a transaction's barrier was published, `close()` could proceed while admitted multi-query reads were in flight; AsyncLocalStorage-based misuse guards misbehaved.

**Root cause** — The single-writer discipline was implicit (ambient async context) rather than an explicit barrier-first queue.

**Fix · [[pi]]** — `ed391c4f0` 2026-09-30 "harden async SQLite adapter queue and close": run calls immediately when idle; transactions publish their barrier before the callback starts; `SqliteStorage.close()` waits for admitted reads and shares one close promise; AsyncLocalStorage guards dropped (pico3 docs `9b2aff2c5` 2026-09-13 had already forbidden AsyncLocalStorage).

**Lesson** — A single-writer storage queue must be explicit and barrier-first; close must drain admitted work.

Related: [[durable-execution]] · [[pi--durable-execution|pi]]
