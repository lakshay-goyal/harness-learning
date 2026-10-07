---
type: absence
harnesses: [codex]
---
# no-cli-process-hardening

The interactive CLI does not ptrace-protect itself, disable core dumps or scrub loader env; only helper daemons (e.g. responses-api-proxy) are hardened.

**What's missing**
- `pre_main_hardening()` exists in `codex-rs/process-hardening/src/lib.rs:12-130` but is used only by helper binaries — `codex-rs/responses-api-proxy/src/main.rs:6` and `codex-rs/voice-host/src/main.rs:49` — not the CLI.

**Evidence of decision**
- Added `d61dea6fe6` 2025-09-25 (`CODEX_SECURE_MODE=1`; "the `codex` process could contain sensitive information in memory, such as API keys"); removed from the CLI 2026-01-08 `d3ff668f68` "remove existing process hardening from Codex CLI (#8951)" because stripping `LD_*`/`DYLD_*` broke spawned commands (issues #8945, #8472) → [[hardening-breaks-child-env]].

**Implication**
- Hardening the agent process hardens everything it spawns; codex accepts same-user observability of the CLI in exchange for an unmodified child environment ([[harness-process-hardening]]).

Related: [[harness-process-hardening]] · [[secret-handling]] · [[harness-credential-leaks-to-tools]] · [[Absences]]
