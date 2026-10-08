---
type: failure
concepts: [subagent-as-subprocess]
harnesses: [pi]
---
**Symptom** — Subagent prompts contained Bun virtual-FS script paths (`/$bunfs/root/…`) from the parent compiled binary (#3002); earlier, child agents were launched via a different `pi` than the running one (#2464).

**Root cause** — Re-invoking yourself as a subprocess used `process.argv[1]` blindly; in a Bun-compiled binary that is a virtual path that leaked into the child's args/prompt; otherwise PATH `pi` could differ from the running build.

**Fix · [[pi]]**
- `7c92bb815` 2026-03-21 — reuse current pi invocation (`process.execPath` + current script) for child agents (#2464, #2465).
- `462b3d21e` 2026-04-15 — skip `/$bunfs/root/` scripts; compiled non-node/bun exec runs `execPath` directly; else fall back to `pi` (`packages/coding-agent/examples/extensions/subagent/index.ts:249-263`) (#3002).

**Lesson** — Re-invoking yourself as a subprocess must resolve the exact running build and must not leak packaging artifacts into the child.

Related: [[subagent-as-subprocess]] · [[harness-package-distribution]] · [[pi--subagent-as-subprocess|pi]]
