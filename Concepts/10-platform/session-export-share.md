---
type: concept
stage: state
tier: candidate
aliases: ["/export", "/share", "export_html", "pi --export", "share viewer", "Radius artifact", opencode export, "--sanitize", ShareNext, opncd.ai]
harnesses: [pi, opencode]
---
Exporting the active branch of a session as a self-contained HTML viewer or JSONL, and sharing it via a hosted artifact or secret gist with a viewer URL.

## Why
- Debugging, bug reports and teaching need a faithful, portable transcript including tool renderings.
- Transcripts contain attacker-influenced content (tool output, web pages) — rendering them as HTML is an XSS surface ([[exported-session-xss]]).
- Sessions are not redacted; sharing can leak secrets (pi tells users to review first).

## Design space
- **Scope**: whole tree vs current branch with re-linked parents (pi JSONL export).
- **Format**: static HTML with embedded data + client-side renderer (pi: base64 session JSON, vendored marked/highlight.js) vs server-rendered page vs plain markdown.
- **Custom tool rendering**: generic JSON vs reuse plugin TUI renderers and convert ANSI → HTML (pi).
- **Hosting**: local file vs public gist vs private gist + viewer URL (pi fallback) vs org-scoped hosted artifact (pi Radius, first choice when signed in).
- **Redaction**: automatic vs user responsibility (pi).
- **Live mirror**: share as a continuously synced copy, key-coalesced 1 s batches, until `unshare` (opencode).
- **Redaction**: opt-in wholesale placeholder redaction of every content field for JSON export (opencode `--sanitize`); none for shares (opencode).
- **Default visibility by context**: share only on public repos in CI (opencode GitHub agent).

## Implementations
- [[pi--session-export-share|pi]] — `/export` HTML/JSONL, `pi --export`, RPC `export_html`; `/share` → Radius artifact or private gist + `pi.dev/session/#<id>`.
- [[opencode--session-export-share|opencode]] — `ShareNext` live mirror to `opncd.ai` (manual/auto/disabled); `opencode export [--sanitize]` JSON + `opencode import`.

## Failures
- [[exported-session-xss]]

## Related
[[session-tree]] · [[extension-ui-primitives]] · [[headless-rpc-mode]] · [[install-telemetry]] · [[client-server-session-split]] · [[secret-handling]] · [[ci-agent-integration]]
