# HalCode9000 — Status

## What This Is

Multi-provider terminal coding agent. Routes to any LLM backend (Anthropic,
OpenAI-compatible APIs, Google, local models) via JSON config + compiled
backend system. Zero Node. Zero Python. ~2.2 MB total across all binaries.

Forked from ClaudeCode on 2026-04-30.

---

## Completed

### Core
- **Multi-provider routing** — Anthropic Messages API, OpenAI Chat Completions,
  Google Gemini. One wire format per backend; internal messages are OpenAI schema.
- **Provider startup menu** — reads `providers/*.json`, populates a selection
  screen. Drop a new JSON file to add a provider — no code changes.
- **7 providers** — Anthropic, OpenAI, xAI/Grok, Google Gemini, DeepSeek, Groq,
  Ollama (local). API keys in `~/.halcode/keys.env`.

### TUI
- Full-screen raw-mode terminal UI with split-pane chrome.
- Bottom-pinned prompt with animated mascot (`─┤ · ├─`), state-driven color
  (orange → red → green).
- **Streaming** — text streams in token-by-token (Anthropic SSE + OpenAI SSE).
- **Scrollback** — chat scroll buffer with PgUp/PgDn.
- **Cursor movement** — left/right, Home/End, insert-at-pos, backspace-at-pos.
- **Bracketed paste** — handled cleanly via `TUI_GetKey` key constants
  (`KEY_PASTE_START`, `KEY_PASTE_END`, `KEY_PGUP`, `KEY_PGDN`, `KEY_ESC`).
- **Reasoning display** — dim italic for DeepSeek `reasoning_content`.
- **Token counter** — `↑input ↓output` updates after every turn.
- **SIGWINCH** handled — resize mid-session.

### Tools (17 total)
| Tool | Status |
|------|--------|
| Read, Head, LS, Write, Edit | Production |
| Bash (30s default, 55s cap) | Production |
| Find, Grep, Git, WebFetch | Production |
| JS (QuickJS VM) | Production |
| MCP (JSONL transport) | Production |
| Skills (20 skill sheets) | Production |
| Agent (sub-agent fan-out) | Production |
| Pgmem (Postgres memory) | Production |
| Relmem (OlympusRepo index) | Production |

All tools are standalone `cc_*_ipc.x` binaries communicating over abstract Unix
sockets (`@halcode/<Name>`). Launched at startup, kept alive until quit.

### Memory
- **Pgmem** — Postgres-backed persistent context (`hc_context`, `hc_files`,
  `hc_symbols`, `hc_sessions`, `hc_tasks`). `park/pickup/search/compact`.
- **Relmem** — OlympusRepo-backed code-aware symbol index. `symbols/callers/calls/graph`.

### Agent orchestration
- **Sub-agents** — the `Agent` tool spawns `HalCode9000.x --agent` children
  with independent history and tool access. Parent orchestrates; no recursive
  sub-agents.
- **Agent waiting** — `Sleep` tool prevents Pgmem deadlock during agent cycles.

### MCP server mode
- `--mcp` flag starts a stdio JSON-RPC server (JSONL transport, protocol `2025-11-25`).
  Exposes all 17 tools to Claude Code sessions. Initialize in ~120ms.

### Session management
- **Persistent logging** — every conversation saved to `~/.halcode/logs/session_<timestamp>.txt`.
- **Project thread memory** — `chat:<proj>:<thread>:turn:<N>` keys in Pgmem.
- **Context pressure** — automatic checkpoint parking at ~65% token window.

### Skills system
- 20 skill sheets across 4 categories (`ailang`, `halcode`, `tools`, `general`).
- `Skills op=list` and `Skills op=read name=<name>` — loaded at session start.
- Served by `cc_skills_ipc.x` daemon reading from `<app_dir>/skills/`.

---

## Known Architectural Constraints

Documented in full in `docs/pain-points.md`. Summary:

1. **Library.JSON hash-collision bug** — `"name"` and `"index"` collide.
   Workarounds: parallel arrays in Anthropic backend, raw string building in
   OpenAI backend.
2. **Arena clobber** — string pointers from auth flow become garbage.
   Fix: integer `kind` field for backend dispatch.
3. **No streaming tool results** — Bash output is fully buffered.
4. **No auth on IPC sockets** — any process on the same WSL2 instance can connect.
5. **History truncation is lossy** — no automatic summarization.
6. **MCP uses JSONL** (not Content-Length framing).

---

## Future

- **Token cost display** — per-turn from usage field × provider pricing.
- **Automatic history compaction** — summarization when context fills up.
- **Olympus tie-in** — pgmem compaction boundaries → Olympus commit/mana annotations.
- **Tool result streaming** — progressive output for long-running Bash commands.
- **IPC auth** — namespace isolation beyond `@` prefix convention.
