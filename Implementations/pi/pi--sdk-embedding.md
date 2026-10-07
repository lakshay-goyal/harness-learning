---
type: implementation
harness: pi
concept: sdk-embedding
commit: b30a6dd77
files: [packages/coding-agent/docs/sdk.md:1-118, packages/coding-agent/src/core/sdk.ts, packages/coding-agent/src/core/agent-session-runtime.ts, packages/coding-agent/examples/sdk/01-minimal.ts, packages/evals/src/harness.ts:337-355]
---
[[sdk-embedding]] in [[pi]].

## Mechanism
- `createAgentSession()` → `{session: AgentSession}` (`packages/coding-agent/docs/sdk.md:7-18`; `core/sdk.ts`). Session owns one conversation, model/tools, queued messages, compaction state, extension runtime (`docs/sdk.md:28`).
- API: `prompt`, `steer`, `followUp`, `abort` (stops + waits idle), `waitForIdle`, `subscribe`, `dispose` (aborts, invalidates extension contexts, removes listeners) (`docs/sdk.md:62-70,56`). `prompt()` resolves after the run incl. automatic retries; `steer()`/`followUp()` return `"queued"`.
- `prompt()` while streaming without steer/follow-up choice **rejects** rather than guessing (`docs/sdk.md:66`) → [[steering-queue]], [[follow-up-queue]].
- Injectable boundaries (`docs/sdk.md:96-110`): `modelRuntime`, `model`, `thinkingLevel`, `scopedModels`; `settingsManager` (merged or in-memory); `sessionManager` (`SessionManager.inMemory()`, `docs/sdk.md:44-50`); `resourceLoader` (`DefaultResourceLoader` with inline `extensionFactories`, or custom `ResourceLoader`); `tools`, `noTools`, `excludeTools`, `customTools`.
- `SessionManager` is authoritative for model context: assigning `session.agent.state.messages` does not replace persisted context (`docs/sdk.md:40-42`) → [[session-tree]].
- `AgentSessionRuntime` adds `newSession()`, `switchSession()`, `fork()`, `importFromJsonl()`; each replaces the active `AgentSession` and recreates cwd-bound services; subscriptions must be rebound (`docs/sdk.md:58-60`; example `13-session-runtime.ts`).
- CLI built-ins (codemode, tool_search, MCP) NOT loaded in SDK sessions; add factories; call `session.bindExtensions()` so MCP connects on `session_start` (`docs/sdk.md:116`) → [[pi--replaceable-builtin-extension]].
- Lower layer also exported: `packages/agent` `Agent` + `agent-loop.ts` (949 lines, in-memory) → [[turn-loop]].
- 14 typechecked examples `examples/sdk/01-minimal … 14-codemode-mcp` (custom model, prompt, skills, tools allowlist, extensions, context files, prompt templates, API keys/OAuth, settings, sessions, full control, session runtime, codemode+MCP).
- Real consumer: pi's eval harness builds sessions via `createAgentSessionServices` + `createAgentSessionFromServices` in-process (`packages/evals/src/harness.ts:337-355`) → [[pi--harness-evals]].

## Constants
- none.

## Evolution
- 2025-12-22 `5482bf3e1` SDK for programmatic `AgentSession`; same day SDK docs (`05e1f31fe`) and project settings factories (`62c64a286`).
- 2026-03-31 `d86122cbd` runtime host for session switching (#2024) → 2026-04-03 `9f9277ccd` closure-based `AgentSessionRuntime`.
- 2026-04-22/23 `1cc303d05`/`f0cf8a59d` replacement-session callbacks + stale ctx errors.

## Evidence commits
`5482bf3e1` `05e1f31fe` `d86122cbd` `9f9277ccd` `1cc303d05` `f0cf8a59d`

## Quirks
- CLI vs SDK default parity intentionally broken for built-in extensions.
- Session replacement silently orphans old subscriptions unless rebound.

## Durable variant (packages/durable)
- `@earendil-works/pi-durable` is itself an embeddable library: open a Harness on Memory/SQLite/JSONL/Cloudflare DO storage, `submit()` with `requestId` dedup, `viewState()`/`watch()` (`packages/durable/README.md:5,94-124,276-302,545`) → [[durable-execution]].

## Failures
[[stale-plugin-context-after-session-replacement]]
