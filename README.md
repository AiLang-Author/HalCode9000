# HalCode9000 Beta v1

```
  ██╗  ██╗ █████╗ ██╗      ██████╗ ██████╗ ██████╗ ███████╗
  ██║  ██║██╔══██╗██║     ██╔════╝██╔═══██╗██╔══██╗██╔════╝
  ███████║███████║██║     ██║     ██║   ██║██║  ██║█████╗
  ██╔══██║██╔══██║██║     ██║     ██║   ██║██║  ██║██╔══╝
  ██║  ██║██║  ██║███████╗╚██████╗╚██████╔╝██████╔╝███████╗
  ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝
```

**The tiniest, most powerful self-hosted agentic coding system.**

370 KB main binary. ~2.2 MB total with all 16 workers.  
Full-screen terminal agent with persistent memory, parallel tool workers, sub-agents, and **Skills** — reusable, version-controlled agent intelligence.

---

## ✨ What Makes It Special

- **Skills System** — Give the agent persistent, composable skills: tool-use patterns, coding standards, architecture rules, domain knowledge, debugging strategies, and more. The model can read, combine, and improve them over time.
- **16 persistent tool workers** over abstract Unix sockets — fast, reliable, and easy to extend.
- **True long-term memory** via PostgreSQL (`pgmem`) + code-aware repository indexing (`relmem`).
- **Sub-agents** — spawn parallel agents with their own tools and context.
- **Multi-provider** — Anthropic, OpenAI, Grok, Gemini, DeepSeek, Groq, Ollama, and any OpenAI-compatible backend.
- **MCP server mode** — drop-in backend for Claude Code and other MCP clients.
- Runs on Linux and WSL2. Zero Node. Zero Python. Zero bloat.

---

## Quick Start

```bash
git clone https://github.com/AiLang-Author/HalCode9000.git
cd HalCode9000
./setup.sh
```

The setup wizard handles PostgreSQL, OlympusRepo, and API key configuration.

Then launch:

```bash
source ~/.halcode/keys.env
./HalCode9000.x
```

---

## Core Features

| Feature | Description |
|---|---|
| **Skills** | Persistent, version-controlled `.skill` files that teach the agent reusable behaviors and knowledge |
| **Multi-Provider** | Switch between providers at startup. Add new ones with a single JSON file |
| **Persistent Memory** | `pgmem` stores sessions, decisions, todos, and context across restarts |
| **Code-Aware Memory** | `relmem` lets the model search symbols and definitions across your entire codebase |
| **Parallel Tools** | 16 standalone workers (Read, Edit, Bash, Git, Grep, WebFetch, etc.) dispatched in parallel |
| **Sub-Agents** | Model can spawn child agents for complex tasks |
| **MCP Mode** | Run as `--mcp` for seamless integration with Claude Code |
| **TUI** | Full-screen terminal interface with live token counters and status |

---

## Skills System

> *The crown jewel.*

Skills turn HalCode9000 into a programmable AI development environment. You — or the agent itself — can create `.skill` files containing:

- Reusable tool-calling patterns
- Project-specific architecture and conventions
- Language-agnostic best practices
- Debugging and refactoring strategies
- Domain knowledge

Skills are loaded dynamically, stored alongside your code, and survive across sessions. The agent can discover, combine, and evolve them over time. This is what makes long-running, deeply contextual work actually practical.

> Example skill files and templates will be added to `/skills/` soon.

---

## Providers

Configured via `providers/*.json` — no code changes needed.

- Anthropic (Claude)
- OpenAI (GPT-4o, etc.)
- xAI / Grok
- Google Gemini
- DeepSeek
- Groq
- Ollama (local)

---

## Tools

All tools are independent binaries communicating over abstract Unix sockets (`@halcode/*`):

`Read` · `Head` · `LS` · `Write` · `Edit` · `Bash` · `Find` · `Grep` · `Git` · `WebFetch` · `JS` · `MCP Client` · `Agent` · `Pgmem` · `Relmem`

Adding a new tool is as simple as writing one binary and registering it.

---

## MCP Server Mode

HalCode9000 ships a dedicated MCP bridge binary (`cc_mcp_ipc.x`) that speaks the Model Context Protocol over stdio. Register it once and any MCP client — Claude Code, Cursor, Windsurf — gets access to DeepSeek, sandboxed Bash, and the Relmem symbol graph.

```bash
claude mcp add -s user halcode9000 -- /path/to/cc_mcp_ipc.x
```

The bridge routes each tool call through HalCode9000's IPC workers without blocking the main session — poll-based multiplexing means Claude Code and HalCode9000 share the same workers concurrently.

---

## Size & Performance

| Component | Size |
|---|---|
| Main binary | ~370 KB |
| All 16 tool workers | ~1.8 MB |
| **Total** | **~2.2 MB** |

Statically linked. Runs fast even on modest hardware.

---

## What's New — Beta v1

HalCode9000 is a native terminal chat client for running LLM agents with a full tool suite — Bash, file I/O, grep, git, web fetch, and more — all dispatched over a lightweight IPC layer written in AILang. No Node, no Electron, no Python runtime. Just a ~500KB binary.

Supports Anthropic, OpenAI, DeepSeek, Gemini, Grok/xAI, OpenRouter, and local Ollama. Add your own provider with a JSON config file.

**Beta v1 changelog:**
- **MCP bridge** — `cc_mcp_ipc.x` is a proper stdio MCP server. Register once with `claude mcp add` and Claude Code (or any MCP client) can call DeepSeek, sandboxed Bash, and Relmem directly
- **Poll-based IPC multiplexing** — tool workers now serve HalCode9000 and external MCP clients concurrently; no blocking, no connection queuing
- **Skills system** — persistent, version-controlled skill sheets teach the agent reusable patterns, project conventions, and domain knowledge across sessions
- **Cursor movement in input** — left/right/home/end keys, insert-at-position, backspace-at-position
- **OpenRouter support** — any model on the OpenRouter catalogue, model sub-menu at startup
- **Session logging** — full conversations including tool calls and reasoning saved to `~/.halcode/logs/`
- **Persistent context via Pgmem** — agents remember working state across sessions
- **Stream stability** — reduced retry waste, turn log preserves completed tool work on drops, 40KB write guard prevents oversized calls
- **UI overhaul** — scroll buffer, split-pane chrome, live token counters, HAL mascot, reasoning display

Bug reports very welcome — open an issue or it didn't happen.

---

## License

MIT — Copyright 2026 Sean Collins, 2 Paws Machine and Engineering.

---

*Ready to go beyond ordinary AI coding tools? Try it. The difference is night and day.*
