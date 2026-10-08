---
type: absence
harnesses: [codex]
---
# no-strict-jsonrpc

The app-server protocol omits the `jsonrpc: "2.0"` field — JSON-RPC-shaped, not compliant.

**What's missing**
- "We do not do true JSON-RPC 2.0, as we neither send nor expect the `"jsonrpc": "2.0"` field" (`codex-rs/app-server-protocol/src/rpc.rs:1-2`).

**Evidence of decision**
- Inherited from the `codex mcp` server it was split from (`d9dbf48828` 2025-09-30).

**Implication**
- Generic JSON-RPC client libraries need adapters; first-party clients use generated TS/Python types ([[codex--client-server-session-split|codex]]).

Related: [[client-server-session-split]] · [[headless-rpc-mode]] · [[sdk-embedding]] · [[Absences]]
