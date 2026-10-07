---
type: failure
concepts: [tool-call-gate, extension-event-hooks, os-level-sandbox]
harnesses: [pi, codex]
---
**Symptom** — A user `!cmd` meant to run remotely (an extension routing `user_bash` to a VM/remote executor) ran on the **local** host when the routing hook threw or returned an invalid result; the failure was logged and execution fell through to the default local shell.

**Root cause** — Hook errors were treated as "no opinion" (fail-open): `undefined`, a throw and a malformed result all meant "use the default executor". For a hook whose purpose is containment, the default is the unsafe path.

**Fix · [[pi]]** — `509ee2bd0` 2026-09-16 (#9662, fixes #9068) "fail closed on user bash hook errors": `emitUserBash` validates results (`{operations}` or `{result}` exactly), reports the error via `emitError` and **rethrows** → command aborted (`packages/coding-agent/src/core/extensions/runner.ts:1261-1290`). Same principle already held for `tool_call`: a throwing handler blocks the tool ("A `tool_call` handler failure blocks the tool as a fail-safe", `packages/coding-agent/docs/extensions.md:260`; `packages/coding-agent/src/core/agent-session.ts:667-680`; `packages/agent/src/agent-loop.ts:770-776`).

**Fix · [[codex]]** — *Not fixed; recorded fail-open design points (observation, not exploited in history):*
- PreToolUse hooks (external processes, Claude-Code protocol) block only on exit 2 with a reason or an explicit JSON block; a spawn error, any other exit code, invalid JSON, or exit 2 without stderr only marks the hook `Failed` and the tool runs (`codex-rs/hooks/src/events/pre_tool_use.rs:200-288`). PermissionRequest hook failures yield no decision and fall through to the normal guardian/user approval — i.e. still gated (`codex-rs/hooks/src/events/permission_request.rs:197-205`). Required Stop hooks with unknown attribution are kept fail-closed (`codex-rs/hooks/src/events/stop.rs:88`).
- Windows elevated setup installs WFP egress filters for the offline sandbox user but "log[s] failures non-fatally" (2026-04-30 `8121710ffe`); non-elevated "no network" is only env-var proxy poisoning (`codex-rs/windows-sandbox-rs/src/env.rs:126-160`).
- Contrast — elsewhere codex fails closed: Guardian review errors → Denied (`codex-rs/ext/guardian-reviewer/src/completion.rs:127-151`), Seatbelt proxy without loopback port (`codex-rs/sandboxing/src/seatbelt.rs:358-364`), proxy DNS errors (`aea82c63ea`), unsafe Linux unreadable globs (`1dac3d9ca0`), process hardening exits 5/6/7 ([[harness-process-hardening]]).

**Lesson** — Policy/containment hooks must fail closed: an error in the guard denies the action, it never falls back to the unguarded default.

Related: [[tool-call-gate]] · [[tool-only-isolation]] · [[extension-event-hooks]] · [[pi--tool-call-gate|pi]] · [[codex--tool-call-gate|codex]] · [[os-level-sandbox]] · [[side-door-input-bypasses-hooks]] · [[network-policy-fail-open-paths]]
