---
type: group
group: 10-platform
---
How a harness is extended, embedded, presented, distributed, operated remotely and evaluated: plugin API and loading, headless/SDK surfaces, TUI, settings, export, telemetry, client/server split, remote execution, self-evaluation.

## Concepts
- [[extension-event-hooks]] — Typed lifecycle event bus where plugins observe/transform/veto at fixed points.
- [[plugin-tools]] — Plugin-registered model-callable tools with schema, renderers, exposure.
- [[runtime-plugin-loading]] — Load uncompiled TS plugins at runtime with host-module aliasing, transactional registration, manual /reload, stale-context invalidation.
- [[replaceable-builtin-extension]] — Minimal core; in-box features implemented on the public extension API and disable/replace-able by name.
- [[harness-package-distribution]] — Bundle extensions/skills/prompts/themes as npm/git packages with manifest.
- [[headless-rpc-mode]] — Long-lived subprocess protocol (JSONL commands/responses/events + UI sub-protocol) for non-JS hosts.
- [[agent-event-stream]] — Typed lifecycle event protocol consumed by TUI/SDK/JSON mode.
- [[sdk-embedding]] — In-process library API with injectable services.
- [[differential-tui-rendering]] — Line-diff terminal rendering + synchronized output; alt-screen transcript.
- [[extension-ui-primitives]] — Mode-portable UI API for plugins (dialogs, widgets, overlays, custom renderers).
- [[layered-settings]] — Global → project → CLI deep merge, locked fields.
- [[session-export-share]] — Export transcript branch to HTML/JSONL; share via gist/hosted artifact.
- [[install-telemetry]] — Opt-out anonymous version ping + vendor-neutral telemetry contracts.
- [[client-server-session-split]] — Agent runs in durable per-session worker process; UIs attach over a protocol via a stable coordinator; relay for remote access.
- [[remote-execution-env]] — Small native daemon on remote host performs fs/exec for local agent over framed stdio (SSH).
- [[harness-evals]] — How the harness evaluates itself: harness adapter, paired A/B arms, state-oracle grading, sandboxed runner, conformance suites.
- [[spec-driven-agentic-development]] — Normative spec + ordered package handoff implemented by agents, reviewed per package.

Failures: [[Platform Failures]]

Related absences: [[no-subagents-core]] · [[no-plan-mode]] · [[no-permission-prompts]] · [[no-sandbox]] · [[no-auto-hot-reload]] · [[no-builtin-mcp-reversed]]
