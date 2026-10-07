---
type: failure
concepts: [remote-execution-env]
harnesses: [pi]
---
**Symptom** — Platform detection failed when a Windows host's default shell mangled the POSIX probe's quoting instead of erroring, so pi-env didn't fall back to the Windows path.

**Root cause** — Fallback only triggered on an `SshError`; garbled-but-successful output wasn't recognized.

**Fix · [[pi]]** — `4bf5a6bc5` 2026-10-05: fall back to the PowerShell `-EncodedCommand` probe on ANY failure except host-key errors; parse output only after a `PI-ENV-PROBE` marker (skips login banners); MINGW/MSYS/CYGWIN results re-probed via PowerShell; `echo %OS%` distinguishes cmd vs PowerShell (`packages/env/src/ssh.ts:288-353`).

**Lesson** — Probe remote platforms with markers and treat any unparseable result as "try the next probe", except security failures.

Related: [[remote-execution-env]] · [[remote-bootstrap-hang]] · [[pi--remote-execution-env|pi]]
