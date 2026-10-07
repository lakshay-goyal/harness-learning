---
type: concept
stage: compaction
tier: candidate
aliases: [serializeConversation, "<conversation>", "[User]:", "[Tool result]:", SUMMARIZATION_SYSTEM_PROMPT, TOOL_RESULT_MAX_CHARS, anti-continuation-instruction]
harnesses: [pi, opencode]
---
For summarization the conversation is flattened into tagged plain text inside one user message, paired with a system prompt forbidding continuation, and per-tool-result text is capped so the summary request itself fits the window.

## Why
- Sent as real chat turns, the summarizer tends to *continue* the conversation (answer the last question, call tools) instead of summarizing (pi: [[summarizer-continues-conversation]], [[summarizer-emits-tool-calls]]).
- The summarizer has the same context window as the agent: serializing full tool outputs made the compaction request overflow — the exact condition compaction is meant to fix (pi: [[summarization-request-overflows]]).
- Role-ordering rules (tool result after call, alternating roles) don't apply to a single text blob, so any history shape can be summarized.

## Design space
- **Chat turns + "summarize the above" user message** (pi v1, OpenCode) vs **serialized text in `<conversation>` tags** ✔ pi (`2add465fb`) vs markdown headings (`# Conversation`, pi split-turn after `d192bd6dc`).
- Anti-continuation system prompt: "Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary." ✔ pi (also reused for bug reports).
- Tool-output budget in serialized input: none ("already token-budgeted", pi `17ce3814a`) → **2000 chars head + "[... N more characters truncated]"** ✔ pi (`c950c692a`). Tool-call args: pi once sliced to 100 chars, now full JSON.
- Include reasoning? pi includes `[Assistant thinking]` since `17ce3814a`.
- Images: dropped (text only) ✔ pi.
- Custom roles converted first (shell runs, summaries → user text) ✔ pi via [[message-conversion-layer]].
- Tool choice: forbid tools via `toolChoice:"none"` (pi tried, broke providers) vs send no tools + reject tool calls ✔ pi.
- **Serialize the kept tail too**: the recent window re-enters context as serialized text inside a `<conversation-checkpoint>` user message, so no provider-native message (signatures, encrypted reasoning) crosses the boundary (opencode v2).
- Extra tags for harness message kinds: `[System update]:`, `[Synthetic context]:`, `[Shell]:` (opencode v2).
- Head replayed as real model messages with `stripMedia` (opencode before `b7f9363393`, broke on orphaned history) → flat text.

## Implementations
- [[pi--transcript-serialization-for-summary|pi]] — `serializeConversation` (`[User]` / `[Assistant thinking]` / `[Assistant]` / `[Assistant tool calls]` / `[Tool result]` ≤2000 chars), `SUMMARIZATION_SYSTEM_PROMPT`, no tools; identical copy in durable.
- [[opencode--transcript-serialization-for-summary|opencode]] — `serialize` with `[User]`/`[Assistant]`/`[Assistant reasoning]`/`[Assistant tool call]`/`[Tool result]` ≤2000 chars inside `<conversation>`, one user message, no tools; v2 also serializes the retained tail.

## Failures
- [[summarizer-refusal]]
- [[summarizer-continues-conversation]]
- [[summarization-request-overflows]]
- [[summarizer-emits-tool-calls]]
- [[domain-biased-summarizer-prompt]]
- [[compaction-request-shape-mismatch]] (opencode: replaying the head as provider messages produced invalid shapes)

## Tradeoffs
- [[compaction-design]]

## Related
[[structured-compaction-summary]] · [[summary-validation]] · [[auto-compaction]] · [[branch-summary]] · [[split-turn-summary]] · [[message-conversion-layer]] · [[tool-output-truncation]]
