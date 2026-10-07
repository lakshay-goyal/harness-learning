---
type: absence
harnesses: [pi]
---
# no-cwd-confinement

**What's missing**
- File tools (read/write/edit/grep/find/ls) accept any absolute path and `../` anywhere. There is no allowlist/denylist and no symlink-escape check.
- `resolvePath` returns `nodeResolvePath(normalized)` for absolute input, else `resolve(baseDir, normalized)` (`packages/coding-agent/src/utils/paths.ts:103-107`; called via `resolveToCwd`, `packages/coding-agent/src/core/tools/path-utils.ts:48-50`).
- `getCwdRelativePath` (`paths.ts:109-118`) computes `isInsideCwd` **only for display**.
- bash has no cwd parameter, but commands can `cd` anywhere.

**Evidence of decision**
- `packages/coding-agent/docs/security.md:19`: "The working folder controls resource discovery and the default location for tools, but it does not prevent commands from accessing other paths available to the Pi process."
- Same posture as [[no-sandbox]]: the boundary is the OS user (`SECURITY.md:11-17`; `docs/security.md:3,15-16`).
- `b172beb92` (2025-11-12): "Full filesystem access - can read, write, or delete anything".
- Logical consequence: confining file tools while bash is unconfined would be "security theater" in pi's terms (`3424550d2`).

**Opt-in replacement**
- `examples/extensions/protected-paths.ts` blocks write/edit to `.env`, `.git/`, `node_modules/` by substring match (`:11,19`) via [[tool-call-gate]]. This is a denylist, not confinement.
- `examples/extensions/sandbox/` gives bash `allowWrite: [".","/tmp"]` and denies reads of `~/.ssh`, `~/.aws`, `~/.gnupg` (`sandbox/index.ts:73-74`). Bash only.
- `tool-override.ts` example: wrap `read` with access control.
- Real confinement: a container or VM, or tool-only isolation ([[tool-only-isolation]], gondolin, [[remote-execution-env]]).

**History**
- What changed is cwd *tracking*, not confinement. Effective cwd became `ctx.cwd || construction cwd` so extension-registered tools follow session cwd changes (`62835ea81`, #8627). Touches bash, edit, find, grep, ls, read and write (`62835ea81` diffstat; e.g. `packages/coding-agent/src/core/tools/bash.ts:271`).
- [[path-normalization]] grew a long list of quirk fixes: `@` prefix (`c9a20a3aa` #1206), `~`, unicode spaces, MSYS/WSL drives, `file://`, macOS screenshot names (`4edb506df` #1078, `9a7bbb283` #181, `d22c120b8`). None added a boundary check.

**Implication**
- A model mistake or an injected instruction can touch any file the user can. Mitigation is entirely external.
- Path-quirk tolerance (normalize instead of reject) is cheap UX precisely because there is no boundary to enforce.

**codex** — *not absent*: OS-level sandbox with writable roots — Seatbelt on macOS, Landlock/seccomp → bundled bubblewrap on Linux (`26f355b67b` 2026-05-05), Windows restricted-token / MXC sandboxes (`codex-rs/linux-sandbox`, `codex-rs/windows-sandbox-rs`, `codex-rs/sandboxing`, `codex-rs/bwrap`, `codex-rs/mxc-sandbox`) → [[os-level-sandbox]]; protected metadata inside writable roots ([[protected-workspace-metadata]]); network egress via a policy proxy (`77222492f9` 2026-01-23) → [[egress-policy-proxy]]. Symlinked roots could escape policy → [[symlinked-roots-escape-sandbox-policy]].

Related: [[path-normalization]] · [[tool-call-gate]] · [[tool-only-isolation]] · [[pluggable-tool-backends]] · [[no-sandbox]] · [[no-permission-prompts]] · [[Absences]] · [[os-level-sandbox]] · [[protected-workspace-metadata]] · [[egress-policy-proxy]] · [[isolation-strategy]]
