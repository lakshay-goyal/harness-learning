---
type: failure
concepts: [tool-call-gate, extension-event-hooks]
harnesses: [pi]
---
**Symptom** — A user `!cmd` meant to run remotely (an extension routing `user_bash` to a VM/remote executor) ran on the **local** host when the routing hook threw or returned an invalid result; the failure was logged and execution fell through to the default local shell.

**Root cause** — Hook errors were treated as "no opinion" (fail-open): `undefined`, a throw and a malformed result all meant "use the default executor". For a hook whose purpose is containment, the default is the unsafe path.

**Fix · [[pi]]** — `509ee2bd0` 2026-09-16 (#9662, fixes #9068) "fail closed on user bash hook errors": `emitUserBash` validates results (`{operations}` or `{result}` exactly), reports the error via `emitError` and **rethrows** → command aborted (`packages/coding-agent/src/core/extensions/runner.ts:1261-1290`). Same principle already held for `tool_call`: a throwing handler blocks the tool ("A `tool_call` handler failure blocks the tool as a fail-safe", `packages/coding-agent/docs/extensions.md:260`; `packages/coding-agent/src/core/agent-session.ts:667-680`; `packages/agent/src/agent-loop.ts:770-776`).

**Lesson** — Policy/containment hooks must fail closed: an error in the guard denies the action, it never falls back to the unguarded default.

Related: [[tool-call-gate]] · [[tool-only-isolation]] · [[extension-event-hooks]] · [[pi--tool-call-gate|pi]] · [[side-door-input-bypasses-hooks]]
