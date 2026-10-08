---
type: failure
concepts: [mcp-integration, process-tree-kill]
harnesses: [opencode]
---
**Symptom** — Local stdio MCP servers (and their own children) survived harness exit; reconnects leaked clients and transports.

**Root cause** — Closing the SDK client killed only the direct child; reconnect replaced clients without closing the old one; failed/timed-out connects left transports open.

**Fix · [[opencode]]**
- `80e1173ef7` 2026-01-13 close existing client before reassignment.
- `c4c0b23bff` 2026-03-01 "kill orphaned MCP child processes and expose OPENCODE_PID on shu… (#15516)": descendants killed (`packages/opencode/src/mcp/index.ts:418-546`, `pgrep -P` walk then SIGTERM).
- `2e6ac8ff49` 2026-03-26 close transport on failed/timed-out connections.

**Lesson** — Own the whole process tree of every server you spawn, including on failed connects.

Related: [[mcp-integration]] · [[process-tree-kill]] · [[bash-descendants-hang-or-lose-output]] · [[opencode--mcp-integration|opencode]]
