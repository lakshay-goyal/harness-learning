---
type: concept
stage: architecture
tier: candidate
aliases: [BashOperations, ReadOperations, EditOperations, FindOperations, ExecutionEnv, NodeExecutionEnv, spawnHook, pluggable-tool-operations, execution-env-abstraction]
harnesses: [pi]
---
Tools keep their model-facing contract but delegate all I/O (exec, read, write, list, canonicalize) to a swappable operations/environment object, so the same tool runs locally, over SSH, in a VM/container, or on a remote daemon.

## Why
- Sandboxing only the tools (not the whole agent) needs a seam below the tool ([[tool-only-isolation]]).
- Remote/durable runtimes need identical semantics on a different host; divergence shows up as subtle output differences (BOM, line counts) unless enforced by differential tests.
- cwd and file identity are environment state, not process state ([[tools-ignore-session-cwd]]).

## Design space
- Tools call `fs`/`child_process` directly (baseline).
- **Per-tool operations interfaces** (pi coding-agent `BashOperations`, `ReadOperations`, …) + spawn hooks.
- **One environment interface for all I/O with typed `Result` errors** (pi durable `ExecutionEnv`), namespaced by filesystem id.
- Remote daemon speaking a framed protocol, conformance + differential tests for parity (pi-env) → [[remote-execution-env]].
- Producer-side output windowing to save bandwidth (pi durable/remote).

## Implementations
- [[pi--pluggable-tool-backends|pi]] — coding-agent `*Operations` seams + `spawnHook`/`commandPrefix`/`shellPath`; durable `ExecutionEnv` (FileSystem & Shell, `Result`, `BinaryReader.scanLines`, `exec({window})`), local Node and remote Rust daemon implementations.

## Failures
- [[tools-ignore-session-cwd]]
- [[read-fails-on-growing-file]]

## Related
[[tool-only-isolation]] · [[remote-execution-env]] · [[shell-execution]] · [[file-read-tool]] · [[per-file-mutation-queue]] · [[durable-execution]] · [[no-sandbox]]
