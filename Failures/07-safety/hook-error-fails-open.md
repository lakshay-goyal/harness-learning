---
type: failure
concepts: [tool-call-gate, extension-event-hooks]
harnesses: [pi, opencode]
---
**Symptom** — A user `!cmd` meant to run remotely (an extension routing `user_bash` to a VM/remote executor) ran on the **local** host when the routing hook threw or returned an invalid result; the failure was logged and execution fell through to the default local shell.

**Root cause** — Hook errors were treated as "no opinion" (fail-open): `undefined`, a throw and a malformed result all meant "use the default executor". For a hook whose purpose is containment, the default is the unsafe path.

**Fix · [[pi]]** — `509ee2bd0` 2026-09-16 (#9662, fixes #9068) "fail closed on user bash hook errors": `emitUserBash` validates results (`{operations}` or `{result}` exactly), reports the error via `emitError` and **rethrows** → command aborted (`packages/coding-agent/src/core/extensions/runner.ts:1261-1290`). Same principle already held for `tool_call`: a throwing handler blocks the tool ("A `tool_call` handler failure blocks the tool as a fail-safe", `packages/coding-agent/docs/extensions.md:260`; `packages/coding-agent/src/core/agent-session.ts:667-680`; `packages/agent/src/agent-loop.ts:770-776`).

**Fix · [[opencode]]** — Oscillation on hook error policy. `070ced0b3f` 2025-12-10 "revert hook try/catch that surpressed errors" (a catch-all had turned hook throws into no-ops); `814a515a8a` 2026-03-24 kept per-hook try/catch only for `config` hooks. At HEAD `Plugin.trigger` runs hooks with no catch, so a throwing `tool.execute.before` fails the call (`packages/opencode/src/plugin/index.ts:284-297`; inferred fail-closed).

**Lesson** — Policy/containment hooks must fail closed: an error in the guard denies the action, it never falls back to the unguarded default.

Related: [[tool-call-gate]] · [[tool-only-isolation]] · [[extension-event-hooks]] · [[pi--tool-call-gate|pi]] · [[side-door-input-bypasses-hooks]] · [[opencode--extension-event-hooks|opencode]]
