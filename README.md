# HalCode9000

**Agentic AI coding assistant for Linux/WSL.** Multi-provider TUI with 15 parallel tool workers, persistent Postgres memory, sub-agent orchestration, and a stdio MCP server mode for Claude Code integration. Zero Node. Zero Python runtime. Built entirely in [AILang](https://github.com/AiLang-Author/Ailang-Self-Hosting-).

```
  ██╗  ██╗ █████╗ ██╗      ██████╗ ██████╗ ██████╗ ███████╗
  ██║  ██║██╔══██╗██║     ██╔════╝██╔═══██╗██╔══██╗██╔════╝
  ███████║███████║██║     ██║     ██║   ██║██║  ██║█████╗
  ██╔══██║██╔══██║██║     ██║     ██║   ██║██║  ██║██╔══╝
  ██║  ██║██║  ██║███████╗╚██████╗╚██████╔╝██████╔╝███████╗
  ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝

    
```

---

## What it does

HalCode9000 is a full-screen terminal coding agent. You type a task; it reasons, calls tools, and works autonomously until the job is done — then hands control back to you.

- **Multi-provider**: DeepSeek, Anthropic, OpenAI, xAI/Grok, Gemini, Groq, Ollama (local). Switch providers at startup from a menu.
- **15 tool workers**: each runs as a separate IPC process, dispatched in parallel after every model turn.
- **Persistent memory**: `pgmem` stores context, symbols, and session history in PostgreSQL — survives restarts, shared across sessions.
- **Code-aware memory**: `relmem` indexes your repositories so the model can answer "where is `X` defined?" across your entire codebase, backed by [OlympusRepo](https://github.com/AiLang-Author/OlympusRepo).
- **Sub-agents**: the `Agent` tool lets the model fan out parallel sub-agents, each with their own tool access and history, reporting back to the parent.
- **MCP server mode**: run as a Claude Code MCP server (`--mcp` flag) so any Claude Code session gets access to all 15 tools via HalCode9000's IPC backend.
- **~800 KB total** — main binary + all 15 tool workers combined, vs ~200 MB+ for the official Node-based Claude Code.

---

## Quick start

```bash
git clone https://github.com/AiLang-Author/HalCode9000.git
cd HalCode9000
./setup.sh
```

The setup wizard will:
1. Install and configure **PostgreSQL** (for `pgmem` persistent memory)
2. Install and start **OlympusRepo** (for `relmem` code-aware memory)
3. Prompt for **API keys** — stored in `~/.halcode/keys.env`, never committed
4. Verify all 16 binaries are present

Then launch:
```bash
source ~/.halcode/keys.env
./HalCode9000.x
```

---

## Providers

Configured in `providers/*.json`. No code changes needed to add a provider — drop a JSON file and it appears in the startup menu.

| # | Provider | Backend | Default model |
|---|----------|---------|---------------|
| 1 | Anthropic | Anthropic Messages API | claude-sonnet-4-6 |
| 2 | OpenAI | OpenAI Chat Completions | gpt-4o |
| 3 | xAI / Grok | OpenAI-compatible | grok-3-mini-fast |
| 4 | Google Gemini | OpenAI-compatible | gemini-2.0-flash |
| 5 | DeepSeek | OpenAI-compatible | deepseek-chat |
| 6 | Groq | OpenAI-compatible | llama-3.3-70b-versatile |
| 7 | Local / Ollama | OpenAI-compatible | (any model you have pulled) |

API keys live in `~/.halcode/keys.env` (set by `setup.sh`). The model never sees them.

---

## Tools

Each tool is a standalone binary (`cc_*_ipc.x`) launched at startup and kept alive as a persistent IPC service. The main binary auto-forks all workers; they die when you quit.

| Tool | Binary | What it does |
|------|--------|--------------|
| `Read` | `cc_read_ipc.x` | Read a file (with offset + limit) |
| `Head` | `cc_head_ipc.x` | First N lines of a file |
| `LS` | `cc_ls_ipc.x` | List directory contents |
| `Write` | `cc_write_ipc.x` | Write or append to a file |
| `Edit` | `cc_edit_ipc.x` | Surgical string replacement in a file |
| `Bash` | `cc_bash_ipc.x` | Run a shell command (30s default timeout, 55s cap) |
| `Find` | `cc_find_ipc.x` | Recursive file search with pattern matching |
| `Grep` | `cc_grep_ipc.x` | Search file contents |
| `Git` | `cc_git_ipc.x` | Git operations |
| `WebFetch` | `cc_webfetch_ipc.x` | HTTP GET (curl-backed) |
| `JS` | `cc_js_ipc.x` | Run JavaScript snippets via AILang's JSVM |
| `MCP` | `cc_mcp_ipc.x` | Call external MCP servers |
| `Agent` | `cc_agent_ipc.x` | Spawn a parallel sub-agent with its own history |
| `Pgmem` | `cc_pgmem_ipc.x` | Postgres-backed persistent memory (binary only) |
| `Relmem` | `cc_relmem_ipc.x` | Code-aware symbol index via OlympusRepo (binary only) |

All tools communicate over abstract Unix sockets (`@halcode/<Name>`), which work correctly in WSL2 without any tmpfs path issues.

---

## MCP server mode

HalCode9000 can run as a Claude Code MCP server, exposing all 15 tools to any Claude Code session:

```bash
# Register once
claude mcp add halcode9000 -- /path/to/HalCode9000/HalCode9000.x --mcp

# Verify
claude mcp list
# halcode9000   ✓ Connected
```

The `--mcp` flag starts the stdio JSON-RPC server (JSONL transport, protocol `2025-11-25`). The initialize handshake completes in ~120ms; tools register in the background over the next ~12s. MCP mode is project-scoped — register it once per project in Claude Code.

---

## Architecture

```
  You
   │
   ▼
  HalCode9000.x          ← entry point, TUI, provider menu, agent loop
   │
   ├── backends/          ← one per wire format
   │   ├── Anthropic.ailang   (Messages API + SSE streaming)
   │   ├── OpenAI.ailang      (Chat Completions, covers 5 providers)
   │   └── Gemini.ailang
   │
   ├── History.ailang     ← ring buffer, pairing-aware eviction
   ├── IPCDispatch.ailang ← fan-out tool dispatcher, 60s per-tool timeout
   ├── Auth.ailang        ← key store + provider credential routing
   └── UI.ailang          ← raw-mode TUI (Library.TUI)
        │
        ▼
  cc_*_ipc.x  ×15        ← tool workers, abstract Unix sockets
        │
        ├── cc_pgmem_ipc.x  → PostgreSQL  (hc_context, hc_symbols, hc_tasks …)
        └── cc_relmem_ipc.x → OlympusRepo (repo index, symbol search)
```

**IPC envelope** — every tool speaks the same JSON protocol over a 4-byte big-endian length-prefixed Unix socket:

```json
// call
{"method":"call","id":"<tool_use_id>","args":{...}}

// result
{"method":"result","id":"<tool_use_id>","ok":true,"content":"...","truncated":false}
```

Adding a new tool: write `cc_<name>_ipc.ailang`, bind `@halcode/<Name>`, add one `RegisterTool` line. No changes to the main binary.

---

## TUI

Full-screen raw-mode terminal UI. Bottom-pinned 5-row prompt:

```
[chat scrollback]
─────────────────────────────────────────────────────  ← top rule
─┤ · ├─ > _                                            ← mascot + input
─────────────────────────────────────────────────────  ← bottom rule
  ↑1234 ↓567    /help · /clear · /quit                 ← token counts + hints
```

- **Prompt rules fill terminal width** — no hardcoded column cap
- **State-driven color**: orange (idle) → red (waiting for model) → green (done)
- **Animated mascot** `─┤ · ├─` with dot-pendulum during model TTFT wait
- **Dim italic** for DeepSeek `reasoning_content` (visible but distinct from answer)
- **Token counter** updates after every turn: `↑input ↓output`
- **SIGWINCH** handled — resize mid-session

---

## Memory system

### pgmem (Postgres)

`~/.halcode/keys.env` contains the connection string set by `setup.sh`. The tool maintains these tables:

| Table | Purpose |
|-------|---------|
| `hc_projects` | One row per project root |
| `hc_files` | Indexed files with mtime + hash |
| `hc_symbols` | Functions, types, variables (FTS via tsvector/GIN) |
| `hc_sessions` | Conversation sessions with token + cost accounting |
| `hc_context` | Parked findings, decisions, todos (replaces CLAUDE.md) |
| `hc_tasks` | Agent task tracking with per-model cost recording |

### relmem (OlympusRepo)

Indexes repositories via an OlympusRepo instance. The model uses `op=symbols` to look up where things are defined across the full codebase — no `find /` needed.

---

## Building from source

Requires the [AILang compiler](https://github.com/AiLang-Author/Ailang-Self-Hosting-) (`ailang.x`).

```bash
# From the AILangSH root:
./ailang.x Applications/HalCode9000/HalCode9000.ailang Applications/HalCode9000/HalCode9000.x

# Tool workers (repeat for each):
./ailang.x Applications/HalCode9000/cc_tools/cc_bash_ipc.ailang  Applications/HalCode9000/cc_bash_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_read_ipc.ailang  Applications/HalCode9000/cc_read_ipc.x
# ... etc
```

Prebuilt x86-64 Linux binaries are included in the repo for convenience.

---

## Binaries

All binaries are statically linked x86-64 Linux ELF executables. They run on:
- Linux (Ubuntu 20.04+, Debian 11+, Fedora 38+)
- WSL2 (Windows 11 / Windows 10 22H2+)

No shared library dependencies beyond libc.

| Binary | Size |
|--------|------|
| `HalCode9000.x` | ~370 KB |
| 15 × `cc_*_ipc.x` | ~100–350 KB each |
| **Total** | **~2.2 MB** |

---

## Private components

`cc_pgmem_ipc.ailang` and `cc_relmem_ipc.ailang` are not included in this repo (proprietary). Compiled binaries are provided. If you want to build from source, contact the author.

---

## License

MIT — Copyright 2026 Sean Collins, 2 Paws Machine and Engineering.

The AILang compiler and standard libraries are in a separate repository under the same license.
