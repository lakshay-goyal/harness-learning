---
type: failure
concepts: [supply-chain-pinning, remote-host-trust, remote-execution-env]
harnesses: [pi]
---
**Symptom** — (Design flaw fixed before release.) The remote execution daemon was deployed as `pi-env-<VERSION>`: two different builds with the same version could collide, a stale or tampered file under the version name would be reused, and trust rested on a version string; old binaries accumulated.

**Root cause** — Identity of an executable shipped to an untrusted/remote host was its name, not its content.

**Fix · [[pi]]** — `b78e6a908` 2026-10-05 SHA-256 verification on deploy; `ed94330a2` 2026-10-05 renamed to content address `~/.pi/mobile/tools/pi-env-<sha256[0:32]>` (`packages/env/src/ssh.ts:370-375`), hash verified with `sha256sum`/`shasum`/`openssl` (refuse if none, `:377-407`) **before every start** (`:480-502`), upload to temp in a 0700 dir → verify → `chmod 700` → `mv -f`, prune other `pi-env-*` (`:434-451`); Windows via base64 + `Get-FileHash` (`:408-432`).

**Lesson** — Name and verify helper binaries by content hash, check before every start, and replace atomically.

Related: [[supply-chain-pinning]] · [[remote-host-trust]] · [[remote-execution-env]] · [[pi--supply-chain-pinning|pi]]
