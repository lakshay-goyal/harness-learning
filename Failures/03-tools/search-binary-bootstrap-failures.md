---
type: failure
concepts: [search-tools, supply-chain-pinning]
harnesses: [pi]
---
**Symptom** — The auto-download of ripgrep/fd (needed by grep/find) failed in several environment-specific ways: startup **crash** on first run; corrupted extraction when rg and fd downloaded concurrently; 404 on Intel macs; "GitHub API error: 403" on **every launch** behind corporate NAT/CI; wrong Termux package name; glibc-linked builds failing on some Linux distros; Debian's `fdfind` not found.

**Root cause** — Harness-managed binary bootstrap depends on network, upstream release assets and platform quirks: `finished(pipe)` missed abort errors and 10 s was too short for multi-MB archives; a fixed extract dir; upstream dropped x86_64-darwin fd builds; anonymous api.github.com quota (60/h) shared behind NAT.

**Fix · [[pi]]**
- `7ddb7c67a` 2026-02-12 (#1433) — Termux package name; Android never downloads (Bionic), tells user `pkg install` (`packages/coding-agent/src/utils/tools-manager.ts:366-372`).
- `757d36a41` 2026-02-25 (#1631) — `PI_OFFLINE=1|true|yes` skips download; network timeouts (`tools-manager.ts:14-18`).
- `3db5715de` 2026-02-26 (#1348) — unique temp extract dir per pid/time/random; Windows zip via System32 `tar.exe` then `Expand-Archive` (`:219-255,286-292`).
- `5c9ce47c5` 2026-03-14 (#2066) — `pipeline()` catches abort errors; download timeout 120 s (`:11-12`).
- `3edb8b5cb` 2026-04-28 — `fdfind` fallback.
- `7afd80d78` 2026-05-16 (#4559) — pin fd `10.3.0` on darwin/x64 (`:265-267`).
- `57e53b0d7` 2026-09-03 (#8708) — latest version from the `releases/latest` **redirect Location header**, not api.github.com (`:104-138`).
- `6aedd1066` 2026-09-03 (#9070) — statically linked musl builds on Linux (`:42,61`).

**Lesson** — Self-bootstrapping dependencies need an offline switch, unique temp dirs, realistic timeouts, no rate-limited APIs, and pins when upstream drops a platform.

Related: [[search-tools]] · [[supply-chain-pinning]] · [[pi--search-tools|pi]]
