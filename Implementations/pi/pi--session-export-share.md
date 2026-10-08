---
type: implementation
harness: pi
concept: session-export-share
commit: b30a6dd77
files: [packages/coding-agent/src/core/session-export.ts:9-29, packages/coding-agent/src/core/export-html/index.ts:160-172, packages/coding-agent/src/core/export-html/tool-renderer.ts:1-6, packages/coding-agent/src/modes/interactive/session-share.ts:56-203, packages/coding-agent/src/config.ts:592-597, packages/coding-agent/src/core/slash-commands.ts:25]
---
[[session-export-share]] in [[pi]].

## Mechanism
- `/export [path]` → HTML (default) or JSONL (`core/slash-commands.ts:25`); CLI `pi --export <session> [out]` (`packages/coding-agent/docs/cli.md:49`); RPC `export_html` ([[pi--headless-rpc-mode]]).
- JSONL export = current branch only: header `{type:"session", version, id, timestamp, cwd}` then each `getBranch()` entry with `parentId` re-linked to the previous one (`core/session-export.ts:9-29`) → [[session-tree]].
- HTML: template with base64-embedded session JSON + client-side rendering with tree sidebar (`core/export-html/index.ts:160-172`; rewrite `256fa575f`), vendored marked + highlight.js; custom tool renderers run as TUI components then ANSI → HTML (`export-html/tool-renderer.ts:1-6`, `ansi-to-html.ts`; `0c6ac4664`, #702). Nested tool-call records shown in exports (`docs/extensions.md:148`).
- XSS hardening: user HTML sanitized, `sanitizeMarkdownUrl` strips C0 controls then allows only `https?|mailto|tel|ftp` schemes for links and images (`export-html/template.js:619-624,1615-1631`), image data/metadata escaped (`ee462dd70`, `6cb23f9b5`) → [[exported-session-xss]].
- `/share` (`modes/interactive/session-share.ts:56-203`): first tries Radius artifact upload (org visibility; needs `radius` OAuth with ≥5 min validity, `:98-107`; "Export the current branch with presentation metadata for Radius", `:49`); fallback `gh gist create --public=false` of the HTML; viewer URL `https://pi.dev/session/#<gistId>` (override `PI_SHARE_VIEWER_URL`) (`config.ts:592-597`).
- Sessions are not redacted; users told to review before `/export`/`/share` (`docs/security.md:93`; `docs/usage.md:82`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_SHARE_VIEWER_URL` | `https://pi.dev/session/` | `src/config.ts:592` |
| Radius share token validity floor | 5 min | `session-share.ts:103` |

## Evolution
- 2025-11-12 `e467a80b5` `/export` self-contained HTML.
- `/share` as secret gist + pi.dev viewer (#380, `packages/coding-agent/CHANGELOG.md:5192`).
- 2026-01-01 `256fa575f` export-html rewrite (tree sidebar, client-side rendering); 2026-01-16 `0c6ac4664` custom tool export rendering (#702).
- 2026-04-22 `ee462dd70` / 2026-06-02 `6cb23f9b5` XSS fixes (#3532, #3819, #3883).
- 2026-08-23 `460191cfc` Radius session shares with context; `session-export.ts` branch serializer.

## Evidence commits
`e467a80b5` `256fa575f` `0c6ac4664` `ee462dd70` `6cb23f9b5` `460191cfc`

## Quirks
- Radius uploads are org-visible; gist fallback is "secret" (unlisted), not private-access.
- Export reuses TUI renderers, so plugin render bugs leak into exports.

## Failures
[[exported-session-xss]]
