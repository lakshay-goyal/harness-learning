---
type: absence
harnesses: [pi]
---
# no-binary-detection-in-read

Weak absence: an observed gap, not a stated decision. Recorded as a finding.

**What's missing**
- The `read` tool sniffs only for **images** (magic numbers in the first `IMAGE_TYPE_SNIFF_BYTES = 4100` bytes: JPEG, PNG excluding animated, GIF, WEBP, BMP; `packages/coding-agent/src/utils/mime.ts:3-23`).
- Any other file is read whole into memory and decoded as UTF-8: `const buffer = await ops.readFile(absolutePath); const textContent = buffer.toString("utf-8");` (`packages/coding-agent/src/core/tools/read.ts:153-157`).
- No NUL-byte sniff, no "binary file" refusal, no size cap before reading. Truncation to 2000 lines / 50KB happens only *after* the full decode ([[tool-output-truncation]]).
- A first line over 50KB returns a hint instead of content: `[Line N is X, exceeds 50.0KB limit. Use bash: sed -n 'Np' path | head -c 51200]` (`packages/coding-agent/src/core/tools/read.ts:179-183`). This caps what the model receives for minified or binary blobs without a newline, but the whole file is still read into memory.
- Related: no ANSI/binary sanitizing of **bash** output on the model path. `stripAnsi` + `sanitizeBinaryOutput` are applied only in rendering (`src/core/tools/render-utils.ts:48`) and for user `!` commands (`bash-executor.ts:78`). `git log -S stripAnsi` on the bash tool → never present.

**Evidence of decision**
- None stated (observed absence; `03-tools`). Consistent with the "plain text output (no line numbers)" read design since `c7a73d4f8` and with minimal tool logic.

**Durable variant** (packages/durable)
- Bounded memory: one `scanLines` pass, then it decodes only enough of the head (> 50 KiB+1 bytes or 2000 newlines, in 64 KiB reads) (`packages/durable/src/tools/read.ts:39-70,133-174`). The same algorithm is ported to Rust for the remote daemon (`packages/env/daemon/src/scan.rs`).
- Images are *refused* with `unsupported_image` (`durable/src/tools/read.ts:114-131`). Other binaries are still decoded with a non-fatal `TextDecoder` (`:185`), so there is still no binary detection; it is just bounded.
- Concurrent-writer guard: a file that shrank or was rewritten during the read is re-read once, then fails with "changed while it was read" (`packages/durable/src/tools/read.ts:84-94`; growing-file fix `cd60a5b99`) → [[file-read-tool]].

**Opt-in replacement**
- `tool-override.ts` example (replace built-in `read`). A `tool_result` hook could detect U+FFFD density and replace the content ([[tool-result-rewriting]]). Neither is shipped.

**Implication**
- Reading a large binary (sqlite db, tarball, model weights) loads it fully into memory. The model then gets up to 50KB of mojibake, which burns context and can confuse it. Open question: any OOM reports? Issues not searched.
- MCP results by contrast save non-image binary embedded resources to temp files (`src/extensions/mcp/tools.ts:124-230`). The binary-awareness exists, just not in `read`.

Related: [[file-read-tool]] · [[tool-output-truncation]] · [[image-normalization]] · [[tool-result-rewriting]] · [[Absences]]
