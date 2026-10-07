---
type: failure
concepts: [harness-process-hardening]
harnesses: [codex]
---
**Symptom** — Pre-main hardening in the interactive CLI stripped all `LD_*` / `DYLD_*` variables (incl. `LD_LIBRARY_PATH`), so every command Codex spawned lost required library paths; users hit broken tools and "significant performance issues" (issues #8945, #8472).

**Root cause** — Hardening scrubbed the *parent's* environment, which every child inherits; the change was "added not in response to a known security issue, but because it seemed like a prudent thing to do" (`d3ff668f68` body).

**Fix · [[codex]]** — 2026-01-08 `d3ff668f68` "remove existing process hardening from Codex CLI (#8951)"; hardening kept only for small credential-holding helpers: `codex-responses-api-proxy` (`codex-rs/responses-api-proxy/src/main.rs:6`), `voice-host` (`codex-rs/voice-host/src/main.rs:49`), Linux proxy helper (`codex-rs/linux-sandbox/src/proxy_lifecycle.rs:119`).

**Lesson** — Hardening the agent process also hardens everything it spawns; scrubbing the parent env is a user-visible behaviour change — isolate secrets in a small hardened helper instead.

Related: [[harness-process-hardening]] · [[secret-handling]] · [[codex--harness-process-hardening|codex]] · [[no-cli-process-hardening]]
