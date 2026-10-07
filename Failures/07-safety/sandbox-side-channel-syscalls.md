---
type: failure
concepts: [os-level-sandbox, egress-policy-proxy]
harnesses: [codex]
---
**Symptom** — Kernel paths outside the obvious filter escaped the sandbox: `io_uring` can create sockets (incl. AF_VSOCK) without a `socket()` syscall; on WSL, AF_VSOCK and `/run/WSL` interop let a Linux-sandboxed command launch Windows processes or re-enter WSL as root; on macOS, `fcntl` `F_MAKECOMPRESSED` (80) / `F_TRANSFEREXTENTS` (110) mutate files through read-only descriptors, bypassing `file-write*` rules.

**Root cause** — The Linux seccomp filter is a denylist (default Allow, `codex-rs/linux-sandbox/src/landlock.rs:280-284`) and Seatbelt rules were written against the common syscall per capability; every alternate path to the same capability stayed open.

**Fix · [[codex]]** — 2026-02-06 `8896ca0ee6` block `io_uring_setup/enter/register` in all modes (`codex-rs/linux-sandbox/src/landlock.rs:195-199`); 2026-09-09 `f71543813f` "Block WSL interop escapes" — deny AF_VSOCK even with network allowed, mask `/run/WSL` (`codex-rs/linux-sandbox/src/landlock.rs:53-62`); 2026-09-18 `04e4d2b40f` "Block mutating fcntls in restricted macOS Seatbelt policies". Also `ptrace`, `process_vm_readv/writev` denied in non-VM modes (`codex-rs/linux-sandbox/src/landlock.rs:183-231`).

**Lesson** — Deny-default must be literal: for every capability you think you removed, enumerate the alternate syscalls/IPC that grant it, and re-audit on each new platform (WSL, new kernel APIs).

Related: [[os-level-sandbox]] · [[egress-policy-proxy]] · [[codex--os-level-sandbox|codex]] · [[network-policy-fail-open-paths]]
