---
type: concept
stage: permissions
tier: variant
aliases: [codex-process-hardening, pre_main_hardening, CODEX_SECURE_MODE, PR_SET_DUMPABLE, PT_DENY_ATTACH, RLIMIT_CORE, disable_process_dumping, anti-ptrace]
harnesses: [codex]
---
Harden the agent's own (or its credential-holding helper's) process against same-user observation and tampering — no ptrace attach, no core dumps, strip loader-injection env vars — because it holds API keys/tokens in memory that sandboxed or co-resident processes could otherwise read.

## Why
- The harness process holds credentials; any same-UID process (including an escaped or unsandboxed tool command) can `ptrace` it or read a core dump.
- `LD_PRELOAD`/`DYLD_INSERT_LIBRARIES` injection runs attacker code inside the credential holder.
- But hardening the parent changes what children inherit ([[hardening-breaks-child-env]]): it is not free.

## Design space
- **None** — pi (credentials at rest 0600 only, [[pi--secret-handling]]).
- **Harden the main CLI** — codex `CODEX_SECURE_MODE` (2025-09), removed from the CLI 2026-01 ([[no-cli-process-hardening]]).
- **Harden only small credential-holding helpers** (✔ codex: `codex-responses-api-proxy`, `voice-host`, Linux managed-proxy helper).
- **Failure mode**: refuse to start if hardening fails (✔ codex exit codes 5/6/7) vs best-effort.
- **Windows**: no-op (codex TODO).
- Root defeats it anyway ("Admittedly, a user with root privileges can defeat these safeguards").

## Implementations
- [[codex--harness-process-hardening|codex]] — `#[ctor]` `pre_main_hardening()`: Linux `PR_SET_DUMPABLE=0` + `RLIMIT_CORE=0` + strip `LD_*`; macOS `PT_DENY_ATTACH` + strip `DYLD_*`; BSD core limit; Windows no-op; used by helpers only.

## Failures
- [[hardening-breaks-child-env]]

## Related
[[secret-handling]] · [[os-level-sandbox]] · [[supply-chain-pinning]] · [[no-cli-process-hardening]]
