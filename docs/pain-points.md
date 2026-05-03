# HalCode9000 — Pain Points

A collected list of non-obvious architectural warts, bugs, and sharp edges. Intended as
pre-work for a future refactoring / hardening session.

---

## 1. The `Library.JSON` hash-collision bug

The AILang JSON library has a bug where `"name"` and `"index"` collide in the same hash bucket.
This forced **two separate workarounds** in production code:

- **Anthropic backend**: Maintains parallel `tool_names[]` / `tool_ids[]` arrays because
  reading `"name"` from a `tool_use` block returns `"index"` instead.
- **OpenAI backend**: `OpenAI_BuildAssistantMsgStr()` constructs the assistant history
  message as a **raw JSON string** (bypassing `Library.JSON` entirely), because
  `tool_calls` would be silently dropped when `reasoning_content` was present.

**Impact**: If you're extending either backend, you'll hit this. Any new field named
`"name"` is suspect.

---

## 2. Arena allocation clobbered the backend selector string

Originally they dispatched backends via `StringCompare(SelectedProvider.backend, "anthropic")`.
This broke because the arena allocator in the auth flow recycled memory, and the string
pointer in `SelectedProvider` became garbage between `CC_MakeProvider()` and `Backend.Init()`.

**Fix applied**: An **integer `kind` field** (1=Anthropic, 2=OpenAI, 3=Gemini). The string
is never read after auth.

**Impact**: If you add a 4th backend, don't rely on string comparison — use the integer
kind field.

---

## 3. Abstract socket namespace is split (`@halcode/` vs `@halcore/`)

12 tools live on `@halcode/*`. Three "core" tools — Agent, Pgmem, Relmem — live on
`@halcore/*`. This implies a different origin (Olympus SDK prebuilts) and possibly
different privilege assumptions.

There's no actual security boundary, but it's a convention you shouldn't break.

---

## 4. No recursive sub-agents

The `Agent` tool spawns `HalCode9000.x --agent`, but the child **cannot** call the `Agent`
tool itself. This prevents fork bombs but means you can't do tree-of-thoughts or
hierarchical decomposition without the parent manually orchestrating.

---

## 5. MCP uses JSONL, not `Content-Length` framing

Almost every other MCP server (including the reference implementation) uses
`Content-Length: N\r\n\r\n` framing. HalCode9000 uses **newline-delimited JSON** because
the AILang JSON parser can't do incremental/chunked parsing.

Claude Code accepts both, but other MCP clients might not.

**Debug note**: During MCP boot, **stdout is redirected to stderr** so the JSONL stream
stays pristine. If you're debugging with `--mcp`, look at stderr for logs.

---

## 6. The WSL2 rules are prompt-only, not enforced

All 12 rules are injected into the system prompt. There is **zero programmatic
enforcement** — the agent can `sudo`, `find /`, or write to `/etc` if the LLM decides to.
The rules are a leash, not a cage.

In sub-agent mode, the rules are re-injected, but a determined model can still violate them.

---

## 7. Tool results are fully buffered (no streaming)

When Bash runs a 30-second command, the entire output is collected in memory, then
returned atomically. There's no progressive streaming of tool output back to the LLM.
Long-running commands block the turn completely.

The "Future Architecture" section acknowledges this as planned work.

---

## 8. No authentication on IPC sockets

Any process on the same WSL2 instance can connect to `@halcode/Read`, `@halcore/Pgmem`,
etc. There's no token, no handshake, no namespace isolation beyond the `@` prefix
convention.

**Impact**: If you're running untrusted code in the same WSL2 environment, it can call
your tools.

---

## 9. `cc_js_ipc.x` runs a full QuickJS VM

The JS tool is a 74KB binary containing an entire JavaScript engine. It has access to
the filesystem and network through AILang's `Library.*` bindings. The system prompt
doesn't sandbox it beyond the WSL2 rules.

Be careful what JS you let the agent execute.

---

## 10. History truncation is lossy

When the context window fills up, old messages are truncated and optionally parked in
Pgmem — but there's **no automatic summarization**. The parked content is raw,
uncompressed, and retrieval relies on the agent remembering to `Pgmem search` for it.

The "auto-compaction" feature is in the future plans, not built.

---

## 11. The SEQPACKET protocol is custom

The 4-byte BE length + JSON body over `SOCK_SEQPACKET` is a clean design, but it's
entirely custom. If you wanted to replace a worker with a Python or Rust implementation,
you'd need to implement this wire protocol from scratch.

There's no OpenAPI spec or gRPC proto — just the AILang source in `IPCDispatch.ailang`.

---

## Summary

The design is solid for a single-developer codebase, but the `Library.JSON` hash bug and
the arena-string-clobber hack are the two things most likely to bite someone modifying the
backends. Everything else is just knowing where the guardrails are soft.
