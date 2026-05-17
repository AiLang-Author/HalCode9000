# HalCode9000 ALPHA v2.0.1

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

```bash
# Register once per project
claude mcp add halcode9000 -- /path/to/HalCode9000.x --mcp
```

Gives Claude Code (or any MCP client) access to all your local tools and memory.

---

## Size & Performance

| Component | Size |
|---|---|
| Main binary | ~370 KB |
| All 16 tool workers | ~1.8 MB |
| **Total** | **~2.2 MB** |

Statically linked. Runs fast even on modest hardware.

---

## License

MIT — Copyright 2026 Sean Collins, 2 Paws Machine and Engineering.

---

*Ready to go beyond ordinary AI coding tools? Try it. The difference is night and day.*
