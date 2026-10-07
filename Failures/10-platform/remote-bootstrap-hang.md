---
type: failure
concepts: [remote-execution-env]
harnesses: [pi]
---
**Symptom** — Remote env setup hung forever: a stuck `ssh` call or daemon start blocked the agent; uploads to Windows hosts never finished.

**Root cause** — No timeouts on bootstrap steps; Windows sshd may never deliver stdin EOF, and PowerShell reads redirected stdin as text, so "cat until EOF" uploads never completed.

**Fix · [[pi]]**
- `6fa21f2a3` 2026-10-05 — 60 s per ssh call, 300 s upload, 60 s daemon start + hello (`packages/env/src/ssh.ts:122-124`; `packages/env/src/connection.ts:16-17`).
- `9a193ff7e` → `b7dfc049e` 2026-10-05 — exact-length read, then base64 lines ending in a `PI-ENV-END` marker read through PowerShell `$input`; hash-verified; rename with 20 × 250 ms retries for scanner/daemon locks (`ssh.ts:408-432`).

**Lesson** — Every network bootstrap step needs a timeout; frame transfers in-band instead of relying on EOF or binary stdin.

Related: [[remote-execution-env]] · [[remote-platform-misdetection]] · [[pi--remote-execution-env|pi]]
