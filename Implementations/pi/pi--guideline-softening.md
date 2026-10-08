---
type: implementation
harness: pi
concept: guideline-softening
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/bash.ts:45-48, packages/coding-agent/src/core/tools/powershell.ts:18-21, packages/coding-agent/src/core/tools/read.ts:20-23]
---
[[guideline-softening]] in [[pi]].

## Mechanism
- No mechanism in code — a prompt-writing practice visible only in history: always-on rules phrased permissively/descriptively instead of imperatively.
- HEAD instances:
  - "You can inspect PI_* environment variables for current model and session details." (`bash.ts:47`, `powershell.ts:20`) — capability statement, not instruction.
  - "Use read to examine files instead of cat or sed." (`read.ts:22`) — keeps the named anti-pattern, drops "You must".
  - "Pi documentation (read only when the user asks about pi itself…)" — conditional scoping instead of "Read it when users ask…" (`system-prompt.ts:162`).
- Remaining imperatives are scoped to a trigger ("When changing multiple separate locations in one file, use one edit call…", `edit.ts:47`) or concern output style ("Be concise in your responses").

## Evolution
| Date | Hash | Before → after | Why |
|---|---|---|---|
| 2025-12-22 | `42d7d9d9b` | "Use read to examine files before editing" → "…**You must use this tool instead of cat or sed.**" | escalation to stop cat/sed reads |
| 2026-03-22 | `235b247f1` | → "Use read to examine files instead of cat or sed." | de-escalation (silent, CHANGELOG `:2914` "Cleaned up `buildSystemPrompt()`") |
| 2026-07-22 | `bb3d7d399` (#6967) | + "Inspect PI_* environment variables for current model and session details." | expose session metadata |
| 2026-08-06 | `4e64de695` (#7128) | → "**You can** inspect PI_* environment variables…" | CHANGELOG `:759` "Softened … in an attempt to reduce unnecessary inspection commands" |
| 2025-11-16 → 2026-01-17 | `0c5cbd006` → `4068bc556` | "Read it when users ask about features…" → "(only when the user asks about pi itself…)" → `b846a4bfc` "read only when…" | scope the docs pointer |
| 2025-11-29 → 2026-05-28 | `186169a82` → `1ab289980` | "Prefer grep/find/ls tools over bash…" removed | rule fired when tools absent (#5132) |

Pattern: escalate wording to fix under-compliance (`42d7d9d9b`), then de-escalate once over-compliance or noise appears (`235b247f1`, `4e64de695`). The "ALL CAPS / NEVER / critical_rule" register was used once (Codex bridge `1650041a6`) and deleted 12 days later (`6484ae279`).

## Evidence commits
`42d7d9d9b`, `235b247f1`, `bb3d7d399`, `4e64de695`, `0c5cbd006`, `4068bc556`, `b846a4bfc`, `186169a82`, `1ab289980`, `1650041a6`, `6484ae279`.

## Quirks
- "in an attempt to" in CHANGELOG — effectiveness of the softening not measured (unverified; no eval cited).
- Softening happens silently inside refactors (`235b247f1`), so it is invisible in release notes.

## Failures
- [[imperative-guideline-over-compliance]]
- [[shell-cat-instead-of-read-tool]]
