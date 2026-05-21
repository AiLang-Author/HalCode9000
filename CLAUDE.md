# Project Memory

## Session 2026-05-20 — cc_mcp_ipc dual-mode + README overhaul

### What was built
- **`cc_tools/cc_mcp_ipc.ailang` — dual-mode rewrite** — when invoked by CC_LaunchTool (argv[2] present), binds `@halcode/MCP` and serves HC's internal model as an IPC daemon. When invoked standalone (no args, via `claude mcp add`), runs as a stdio NDJSON MCP server routing tools to @halcode/Agent, Bash, Relmem, Pgmem. Fixes the startup hang where HC would block trying to connect to @halcode/MCP (which the old stdio-only version never bound).
- **`cc_tools.json`** — restored `cc_mcp_ipc.x` entry (had been incorrectly removed as a "fix" for the hang). Entry is back and correct.
- **argv detection pattern** — `CCMcp_GetArg(n)` reads `/proc/self/cmdline` (null-separated). Same pattern as `CCSkills_GetArg` in cc_skills_ipc. Use this anywhere a tool needs to branch on whether CC_LaunchTool launched it.
- **README.md** — DeepSeek rewrote the marketing copy (Beta v1 framing, structured changelog, confident tone). Then corrected: 23 tools (not 16/17), ~490KB main binary (not 370KB), ~3.8MB total (not 2.2MB), 24 skill sheets (not 20), tools list updated to match cc_tools.json exactly.

### Key facts (current)
- **Tool count**: 23 registered in cc_tools.json
- **Main binary**: ~490 KB (HalCode9000.x = 501,191 bytes)
- **All 23 tool workers**: ~3.3 MB
- **Total**: ~3.8 MB
- **Skill sheets**: 24 in `/skills/` (ailang×14, general×3, halcode×4, tools×3)
- **MCP registration**: `claude mcp add -s user halcode9000 -- /path/to/cc_mcp_ipc.x` (no --mcp flag — cc_mcp_ipc.x IS the MCP server)

### Poll-based IPC multiplexing (load-bearing)
cc_agent_ipc, cc_bash_ipc, and cc_relmem_ipc all use SYS_POLL=7 in their inner RecvMsg loops so they can serve HC's persistent connection AND single-shot connections from cc_mcp_ipc (stdio mode) concurrently. Do not remove this — it's what makes the MCP bridge work without blocking.

### Next priorities
- **cc_relmem_ipc recompile** — blocked on Library.Hash InlineAsm migration (compiler dev session ongoing). Binary in repo is pre-migration.
- **Soul/identity system** — SOUL.md + pgmem-backed framework rows. Design settled, not built.
- **Skill quality pass** — tighten trigger conditions, output format, stop conditions on existing sheets.

## Session 2026-05-29 — Skills system + MCP registration

### What was built
- **`cc_tools/cc_skills_ipc.ailang`** — new Skills tool. `op=list` finds both `skill-name/SKILL.md` dirs and flat `skill-name.md` files. `op=read name=<n>` tries directory format first, falls back to flat. Name/socket taken from argv[1]/argv[2] (passed by CC_LaunchTool), not hardcoded.
- **`CC_LaunchTool`** updated — now accepts `tool_name` + `tool_socket` params and passes them as argv[1]/argv[2] to all tool binaries. Old tools ignore them safely.
- **`build.sh`** updated — added loop for `stat`, `wc`, `du`, `diff`, `olympus`, `sleep`, `skills`.
- **System prompt rule (18)** — "at session start call `Skills op=list`, load relevant sheets with `Skills op=read name=<skill>`."
- **`~/.halcode/skills/ailang-dev/SKILL.md`** — starter skill covering AILang patterns, syscall table, WSL2 rules, build commands.
- **Anthropic skills repo** cloned at `skills/` subdirectory — `skills/skills/*/SKILL.md` format confirmed.
- **Symlink fixed**: `AILangSH/Applications/HalCode9000` → `Documents/HalCode9000`.
- **MCP registered** user-scope: `claude mcp add -s user halcode9000 -- .../HalCode9000.x --mcp`

### Next priorities
- **Improve skill quality** — the prompts are rough; DeepSeek over-generates and wanders without tight skill sheets. Each skill needs: clear trigger conditions, strict output format, explicit stop conditions.
- **DeepSeek control** — add a `deepseek` skill sheet that sets tone/format rules for DS specifically (concise, no preamble, stop after task complete, max tool chains).
- **Soul/identity system** — SOUL.md + pgmem-backed framework rows (`type='soul_framework'`). Design is settled, not yet built.
- **cc_skills_ipc not yet compiled at time of restart** — run `./build.sh --tools-only` from AILangSH after restart to get `cc_skills_ipc.x`.
- **Skills dir to populate**: `~/.halcode/skills/` — currently only `ailang-dev`. Add `deepseek-agent`, `pgmem-queries`, `wsl-debug` at minimum.

## Hard Rules

- **NEVER read image files** (PNG, JPG, JPEG, BMP, GIF, ICO, SVG, WebP, TIFF, TGA, TVG, etc.) with the Read tool. This causes crashes. No exceptions.

## Architecture Notes

- **AKContext system:** Explicit `LinkagePool.AKContext` handles. Each context (main window, toolbar, deskbar, menu, dialog) owns its own node buffer, extra table, and event state. `AK_CreateContext()` allocates, all AK_* functions take `ctx` as first param.
- Toolbar actions fire on UP (not DOWN). Action string -> EventRouter queue -> `EventRouter_Drain` in main loop dispatches.
- `Menu_Show` creates its own AKContext, builds tree, renders to surface, destroys context. Surface stored in MenuState. `Menu_Blit` called from `Win_BlitAll`.
- Main loop: Evdev_Poll -> DrainInput -> Win_RenderDirty -> EventRouter_Drain -> IPCBroker_Poll -> Deskbar_Refresh -> DebugLog_Render -> Win_BlitAll -> sleep(16ms).
- Deskbar has its own AKContext stored in `DeskbarState.ak_ctx`. No global swap needed.
- Each window toolbar has its own AKContext stored via `WinMgr_SetToolbarCtx(idx, ctx)`.
- **IPC Broker** (`Display/IPC/Library.IPCBroker.ailang`): Embedded in display server. Unix socket at `/tmp/ailang_display.sock`. Non-blocking `poll(0)` once per frame. 8-client max. Protocol: 4-byte BE length prefix + JSON. Methods: `register`, `window.create`, `window.update` (app->server); `window.created`, `window.closed`, `input.action`, `input.key`, `input.mouse` (server->app).
- **Start Menu** (`Display/Menu/Library.StartMenu.ailang`): Windows XP/7-style popup panel above deskbar. Own AKContext, own surface, positioned overlay. Lists services from PostgreSQL cache + system items.
- **EventRouter action routing**: System actions (`win.`, `app.`, `menu:`, `sys.`, `fd.` prefixes) always handled locally. Non-system actions from IPC-owned windows forwarded to app via `IPCBroker_RouteAction`.
- **Init sequence**: `SysDisplay_Init -> EventRouter_Init -> Dialog_Init -> Menu_Init -> Deskbar_Init -> IPCBroker_Init -> StartMenu_Init -> HTML_Init -> PageSurface_Init -> Doc_Init`

### Compiler Constraints

- **6-arg limit**: SysV AMD64's 6 register args (RDI, RSI, RDX, RCX, R8, R9) with no spill. `analyzer.x` arity checker enforces this.
- **StoreValue**: Defaults to 8-byte (qword) writes. Use `StoreValue(addr, val, "dword")` for 4-byte writes.
- **MemoryCopy/MemorySet**: Emit `CLD` + `REP MOVSB/STOSB` with register save/restore.

### Headless Testing

`FB_InitHeadless(w, h)` allocates anonymous mmap buffer instead of `/dev/fb0`. Test binaries override `RenderFB_InitDouble` to call `FB_InitHeadless(1920, 1080)`.

### HTML Toolbar System

`toolbar=` attribute on `<window>` tag: `"none"` (0), `"about"` (1, default), `"file"` (2), `"full"` (3). `Win_BuildAppToolbar(ctx, mode, app_title)` builds the tree.

## Key Subsystems (Condensed)

### Shared Memory Canvas

Zero-copy pixel streaming via `/dev/shm/ailang_canvas_<win_id>` (`MAP_SHARED`, BGRA). IPC messages: `canvas.attach`, `canvas.present`, `canvas.detach`. Per-window `CanvasState` (48-byte entries): ACTIVE, SHM_PTR, SHM_SIZE, SURF, MOUSE_CAPTURE, DIRTY fields.

### Xvfb Sandboxed Apps (Chrome, VS Code, Ladybird)

3-process stack: Xvfb (virtual X, `-fbdir` for mmap framebuffer) -> app (`--display=:N`) -> xdotool (persistent stdin pipe for input). Direct mmap of Xvfb framebuffer file (xwd format, 3232-byte header offset). Row-by-row viewport copy to ShmCanvas each tick.

- Chrome: Xvfb :99, fbdir `/tmp/chrome_fb/`, profile `/tmp/chrome_ailang_profile`, `--start-maximized`. Do NOT use `--disable-software-rasterizer`.
- VS Code: Xvfb :98, fbdir `/tmp/vscode_fb/`, profile `/tmp/vscode_ailang_profile`, `--maximize`, `--new-window`.
- Ladybird: Native IPC client (no Xvfb needed) — uses Ailang's ShmCanvas directly via C++ integration in `~/ladybird/UI/Ailang/`.
- PID file system for cleanup (`/tmp/*_ailang_{xvfb,browser,xdotool}.pid`). `DropPriv()` before execve (stat `/home/bob` for uid/gid).
- MOUSE_CAPTURE flag for VM-style mouse forwarding. Mouse move coalescing (one xdotool per tick).
- Persistent Xvfb on resize — no process restart, just ShmCanvas recreate + `xdotool windowsize`.

### Audio Engine

Direct ALSA (`/dev/snd/pcmC0D0p`), S16LE 48kHz stereo. 3-bus mixer (app/system/master). Audio-driven frame sync for video. Volume 0-1024 (256=unity). Replay: must call `Audio_Prepare()` after `Audio_Drop()`.

### Terminal Emulator

PTY + VT100 state machine (NORMAL/ESC/CSI/OSC). 8x16 bitmap font (`Library.TermFont.ailang`). Grid: 4-byte codepoints + BGRA fg/bg per cell. Truecolor (256-color + 24-bit RGB). DEC private modes (?1049/?25/?7/?1). Scrollback ring buffer (1000 lines). UTF-8 multi-byte decoder. Dynamic resize via `TIOCSWINSZ`.

## Library Directory Structure

Import paths use dots: `LibraryImport.Display.Window.WinManager` -> `Librarys/Display/Window/Library.WinManager.ailang`.

```
Librarys/
├── Library.{Arena,XArrays,StringUtils,JSON,HashMap,Socket,ShmCanvas,KeyMap,TextBuffer,TermFont,TUI,Math}.ailang
├── Compiler/                       # Compiler subsystem
├── AIMacro/                        # Macro subsystem
├── Display/                        # Display server
│   ├── System/    # SysDisplay, EventRouter, Screenshot
│   ├── Window/    # WinManager, WinToolbar, WinInput, WinStack, WinRender
│   ├── Input/     # DInputTypes, DInputEvdev, DInputDiscover, Cursor, CursorBitmap
│   ├── UI/        # Auckland, AucklandEvent, AucklandBind, TextRegion, PaneDecorator, Dialog, AboutDialog, FileDialog, NotepadApp
│   ├── Menu/      # Menu, StartMenu, CascadeMenu, Deskbar
│   ├── Render/    # Framebuffer, DRenderFB, DSurface*, DCompose*, DRing*, DZone*, Fonts, VIF, VIcon, AudioEngine
│   ├── Content/   # Document, PageSurface, HTMLParse, Editor
│   ├── Theme/     # UIConfig, UIScale, UITheme
│   └── IPC/       # IPCBroker, InputRouter
├── Browser/                        # JS engine + HTML browser
│   # JSLexer, JSParser, JSCompiler, JSRuntime, JSVM, JSBridge, JSEngine
│   # HTMLTokenizer, HTMLDom, CSSParse, HTMLLayout, HTMLRender
└── DnD/                            # D&D RPG game
    ├── Engine/    # DND, GameConfig, World, Portal, Encounter, DICE
    ├── Character/ # Character, Item, EquipScreen
    ├── Battle/    # BattleScreen
    ├── Commerce/  # Shop, Inn
    ├── Save/      # Save, SaveScreen
    └── Web/       # HTMLBroadcast, DND_HTML_Output_engine
```

IPC apps only import generic root libs — no Display/ imports.

## PostgreSQL Services

```sql
CREATE TABLE IF NOT EXISTS services (
    id SERIAL PRIMARY KEY, name TEXT NOT NULL UNIQUE,
    binary_path TEXT, args TEXT, autostart BOOLEAN DEFAULT false,
    restart_policy TEXT DEFAULT 'never', depends_on TEXT,
    run_as TEXT DEFAULT 'nobody', priority INTEGER DEFAULT 50,
    enabled BOOLEAN DEFAULT true, encryption_key_id INTEGER,
    display_name TEXT
)
```

Seeded: notepad, files, calculator, grep, canvas_demo, videoplayer, terminal, claude, chrome, ladybird.

## Build & Run

```
./ailang.x Main.ailang SysDisplay.x                        # display server
./ailang.x Applications/calc_ipc.ailang calc_ipc.x         # calculator
./ailang.x Applications/grep_ipc.ailang grep_ipc.x         # grep
./ailang.x Testcode/canvas_demo.ailang canvas_demo.x       # canvas demo
./ailang.x Applications/videoplayer.ailang videoplayer.x   # video player
./ailang.x Applications/terminal_ipc.ailang terminal_ipc.x # terminal
./ailang.x Applications/claude_ipc.ailang claude_ipc.x     # claude code
./ailang.x Applications/chrome_ipc.ailang chrome_ipc.x     # chrome browser
./ailang.x Applications/vscode_ipc.ailang vscode_ipc.x    # VS Code
./ailang.x Applications/ladybird_ipc.ailang ladybird_ipc.x # Ladybird browser (native IPC)
./ailang.x dnd_game.ailang dnd.x                           # DnD game
./ailang.x Calc.ailang Calc.x                              # calc unit tests
./ailang.x TestCode/test_main.ailang test_main.x           # headless tests (125 steps)
./ailang.x TestCode/test_js_e2e.ailang test_js_e2e.x       # JS engine E2E tests
./ailang.x TestCode/bench_js.ailang bench_js.x             # JS benchmark (fib)
./ailang.x Applications/browser_ipc.ailang browser_ipc.x   # Ailang native browser
./SysDisplay.x                                              # run on TTY (Ctrl+Alt+F2)
```

### Kernel Module Path (`-kmod`)

`./ailang.x -kmod source.ailang ail_payload.o` produces ET_REL. Drop into `kernel_module/shim/`, `make`, `sudo insmod ail_combined.ko`. Steps 1-5 done, step 6 (insmod test) pending dedicated Linux box.

## Ladybird Browser Integration

Native IPC client (no Xvfb sandboxing needed). C++ integration in `~/ladybird/UI/Ailang/` (10 files, ~1100 lines):
- `AilangIPC.h/cpp` — socket client, JSON protocol, ShmCanvas management
- `Application.h/cpp` — extends `WebView::Application`, IPC message dispatch, toolbar actions (`lb.back`/`lb.fwd`/`lb.reload`)
- `WebContentView.h/cpp` — `ViewImplementation` for Ailang backend, BGRx8888->BGRA paint, keyboard/mouse mapping
- `Events.cpp` — evdev scancode -> `Web::UIEvents::KeyCode` (95 mappings)
- `main.cpp` + `CMakeLists.txt`

Window config: `config/ladybird.html` (1024x700, `toolbar="about"`).
Build: `~/ladybird/Build/release/bin/Ladybird` (compiled, 2.4MB).

## JavaScript Engine (Phase 6)

Bytecode VM architecture: `<script>` source -> JSLexer (tokenize) -> JSParser (recursive descent AST) -> JSCompiler (AST -> bytecode + constant pool) -> JSVM (fetch-decode-execute) -> JSRuntime (values, coercion, built-ins) -> JSBridge (DOM bindings) -> JSEngine (orchestrator).

7 libraries in `Librarys/Browser/`:

| Library | LOC | Role |
|---------|-----|------|
| JSLexer | ~900 | Tokenizer, ~45 token types |
| JSParser | ~1100 | Recursive descent, Pratt precedence, ~35 AST node types |
| JSCompiler | ~1100 | AST -> bytecode (~50 opcodes), constant pool, local resolution |
| JSRuntime | ~1000 | JSValue (16-byte tagged: type+payload), coercion, object/array ops |
| JSVM | ~1100 | Branch-dispatch loop (O(1) opcode dispatch), 4096-deep value stack |
| JSBridge | ~900 | DOM bindings: getElementById, innerHTML/textContent, addEventListener, console.log, setTimeout |
| JSEngine | ~700 | Orchestrator: extract `<script>` tags, lex->parse->compile->run pipeline |

**Value types**: UNDEFINED(0), NULL(1), BOOLEAN(2), NUMBER(3, 64-bit signed int), STRING(4), OBJECT(5, XSHash), FUNCTION(6), ARRAY(7, XArray). Integer-only for v1 (no floats). No GC — arena per-page lifetime.

**Key patterns**:
- `JSCompDot` FixedPool for MEMBER_DOT assignment compilation (survives recursive JSComp__CompileExpr calls)
- `JSBridgeTmp` / `JSVMTmp` FixedPools for clobber-safe temporaries across function calls
- `JSRT_ToString` returns JSValue pointer; use `JSRT_GetPayload()` to extract raw C string
- `JSBridge__GetDomIdx` checks `__dom_idx` property on JS wrapper objects to map back to DOM nodes

**Test suites**: `test_js_e2e.ailang` (9 tests, 31/32 assertions passing — variable-to-innerHTML assignment WIP), `bench_js.ailang` (7 micro-benchmarks, 5.9x faster than V8 overall; fib(20) 41x, arith 89x).

**Status**: String literal innerHTML mutation works end-to-end. Variable-to-innerHTML assignment has a stack ordering issue in SET_PROP dispatch (obj resolves as NUMBER instead of OBJECT). Under investigation.

**Build**: `./ailang.x TestCode/test_js_e2e.ailang test_js_e2e.x && ./test_js_e2e.x`

## Pending Work

- **JS engine innerHTML variable assignment** — SET_PROP stack ordering bug when RHS is a variable (obj pops as NUMBER not OBJECT); string literal path works
- **Ladybird live testing** — test `ladybird_ipc.x` on live display server, performance tuning, tab management
- **Terminal polish** — toolbar actions, cursor blink, mouse reporting (?1000h/?1006h)
- **Audio engine split** — extract from display server into standalone service
- **Video player seek** — FF/RW via command pipe or restart-with-offset
- **Scientific calculator** — trig, log, parentheses
- **Start Menu UI** — side navigation, categories, running-app indicators
- **Encryption at rest** — login gates master key, per-service keys
- **SSE2 optimization** — FB_ClearBuffer, compiler integer SSE2 emit

---

## HalCode9000 — Native AILang Chat Client

`Applications/HalCode9000/` — terminal-mode chat client against multiple LLM backends. All binaries live in the HalCode9000 folder alongside the source.

### Build commands

```
cd /mnt/c/Users/Sean/Documents/AILangSH
./ailang.x Applications/HalCode9000/HalCode9000.ailang Applications/HalCode9000/HalCode9000.x
./ailang.x Applications/HalCode9000/cc_tools/cc_bash_ipc.ailang     Applications/HalCode9000/cc_bash_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_read_ipc.ailang     Applications/HalCode9000/cc_read_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_write_ipc.ailang    Applications/HalCode9000/cc_write_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_ls_ipc.ailang       Applications/HalCode9000/cc_ls_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_head_ipc.ailang     Applications/HalCode9000/cc_head_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_webfetch_ipc.ailang Applications/HalCode9000/cc_webfetch_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_pgmem_ipc.ailang    Applications/HalCode9000/cc_pgmem_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_relmem_ipc.ailang   Applications/HalCode9000/cc_relmem_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_find_ipc.ailang    Applications/HalCode9000/cc_find_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_grep_ipc.ailang    Applications/HalCode9000/cc_grep_ipc.x
./ailang.x Applications/HalCode9000/cc_tools/cc_git_ipc.ailang     Applications/HalCode9000/cc_git_ipc.x
cd Applications/HalCode9000 && ./HalCode9000.x
```

HalCode9000.ailang imports backends/ and UI.ailang transitively — a single compile of HalCode9000.ailang rebuilds everything except the cc_tools.

### Provider menu (startup)

```
1. Anthropic   (claude-sonnet-4-6)
2. OpenAI      (gpt-4o)
3. Grok        (xAI) — grok-3-mini-fast
4. Gemini      (gemini-2.0-flash)
5. Local       (localhost:11434, ollama)
6. DeepSeek    (deepseek-v4-flash)
7. OpenRouter  (model sub-menu: a-e presets + m manual + q back)
--- User Connections ---
8+. Loaded from ~/.halcode/connections/*.json at startup
a.  Add connection (interactive: backend type, name, model, base URL, key hint)
q.  Quit
```

**OpenRouter** uses the OpenAI backend (pure OpenAI-compat). `base_url = https://openrouter.ai/api`, `api_path = /v1/chat/completions`. `OPENROUTER_API_KEY` env var is checked automatically. In-chat `/model` hint shows OpenRouter model IDs when OpenRouter is active.

**Grok** (`api.x.ai`) — option 3, NOT Groq (different company). `XAI_API_KEY` env var is checked automatically.

**Custom connections** are stored as JSON files in `~/.halcode/connections/conn_<timestamp>.json`. Built with raw `StringConcat` (NOT `Library.JSON`) to avoid the known XSHash collision bug that corrupts `key_hint` and `default_model` on write. Loaded via `Backend.LoadProviders()` at menu startup; re-scanned after a new connection is added.

### UI layout (5-row prompt, as of 2026-04-30)

```
[chat scrollback region]
 ─────────────────────────  ← top rule (straight ─, no ╭/╰)
 > input here               ← body row (1 row)
 ─────────────────────────  ← bottom rule
   ↑1234 ↓567   /help · /clear · /quit   ← hint row (tok_in/tok_out left, commands right-aligned)
```

- `UILayout.prompt_h = 5` (quote + top_rule + body + bot_rule + hint)
- `UI.SetTokens(in, out)` — stores to UILayout.tok_in/tok_out, repaints hint row
- `UI.SetQuote(text)` — paints a dim status/quote line above the prompt box
- `UI.AnimTick()` — ticks mascot animation during model TTFT wait (call from idle poll loop)
- `UI.ChatPrintDim(s)` — dim+italic print for DeepSeek reasoning_content stream

### Stream-drop stability fixes (2026-05-03)

Three changes that work together to reduce the cost and corruption of mid-stream API drops:

**1. CC_TurnLog** (`HalCode9000.ailang`) — on every completed tool call, appends `[ToolName] arg_summary\n<first 4096 bytes of content>\n---\n` to `/tmp/hal_turn_log.txt` (O_APPEND, mode 0644). Preserves completed work even if the stream drops before the model finishes the turn. Error message on final retry now tells the user to check this file.

**2. max_retries reduced 5 → 3** — cuts wasted API spend by 40% on persistent drops. The turn log makes retry reduction safe.

**3. Write tool 40KB size guard** (`cc_tools/cc_write_ipc.ailang`) — if `content` exceeds 40,000 bytes, returns an error immediately with the byte count and a message telling the model to split into ≤500-line chunks. Prevents oversized Write calls from triggering stream drops in the first place.

### UI corruption fixes after stream drop (2026-05-03)

Three bugs caused visual corruption when a stream dropped mid-turn:

**Bug 1 — orphaned "⚙ Preparing tool:" line** (`backends/OpenAI.ailang`): When the first `tool_use` delta arrived during SSE streaming, `UI.ChatTag("⚙", "Preparing tool: <name>", 8)` was emitted. On stream drop this line was left in chat with no corresponding tool result. **Fix**: removed the `UI.ChatTag` call entirely — `UI.ToolCallStart` already handles tool-start display.

**Bug 2 — animation characters bleeding into prompt frame** (`HalCode9000.ailang`): After a stream drop `ui_state` stayed at 1 (waiting/streaming), so `UI.AnimTick()` kept painting mascot chars `▅▇ ⠸ ▃▇` and the elapsed timer into prompt frame rows during the retry `CC_SleepMs` delays. **Fix**: added `UI.SetState(0)` immediately after `UI.ChatTag("✗", "API connection dropped...")`. Each retry re-enters state 1 via `UI.SetState(1)` at the top of the retry loop — so the fix only affects the inter-retry idle period.

**Bug 3 — reasoning content overflowing top rule bracket**: When `ui_state = 1` and `UILayout.reasoning_str` contains large DeepSeek thinking content, `UI_PaintTopRule` renders it in the `┌─[ ... ]──` bracket. Long strings overflow the terminal line width, pushing all subsequent rows down and causing mascot chars to appear in the bottom rule. **Fix**: same `UI.SetState(0)` call — with `ui_state != 1`, `UI_PaintTopRule` no longer shows reasoning content.

### Known UI.ailang issue to never repeat

The Write tool wrote a literal `\n` (backslash-n, 0x5c 0x6e) at the end of UI.ailang as part of a test marker comment. The AILang lexer saw `\` at column 1 as "Unknown character" and refused to compile. Fixed by trimming the trailing garbage bytes. **Never append `\n` as literal text to .ailang files** — it must be an actual newline byte.

### DeepSeek tool_calls fix (backends/OpenAI.ailang)

Library.JSON's XSHash dropped `tool_calls` when `reasoning_content` was also present in the same object (root cause unclear — bucket collision or ordering). Fix: `OpenAI_BuildAssistantMsgStr()` builds the entire assistant message as raw JSON via `StringConcat` + `JSON.EscapeString`, then `ParseJSON` back. Bypasses XSHash for that object entirely.

**Critical**: OpenAI `arguments` field must be a JSON-encoded STRING (not inline object): `"arguments": "{\"path\":\"/etc/hostname\"}"`. Use `JSON.EscapeString(args_ptr)` before inserting.

### Token display

Both Anthropic and OpenAI backends now call `UI.SetTokens(in, out)` after each turn instead of printing dim text to chat. Anthropic reads `message.usage.input_tokens` from `message_start` event, `usage.output_tokens` from `message_delta` event.

### Relmem (cc_relmem_ipc)

- Index at `~/.claude/relmem/index.json` (~4MB, already built)
- Socket: `@halcode/Relmem` (abstract Unix socket, bypasses WSL2 tmpfs)
- `Op_Index` uses model-interaction pattern: model enumerates files via Bash `find`, then passes `files=<newline-separated-paths>` to op=index for grep-style symbol extraction. Broad-path guard still in place (rejects `/`, `/mnt`, `/mnt/c*`, `/home`).
- Use `op=symbols` to query the built index — never `op=index` with broad paths.

### WSL2 Hard Rules (system prompt rules 1-5)

Encoded in `CCConst.SYSTEM_PROMPT` in `HalCode9000.ailang`:
1. NEVER `find /`, `/mnt`, `/mnt/c` — unbounded, hangs permanently
2. Use `Relmem op=symbols` to locate files in the indexed codebase
3. If using `find`, scope to a specific known subdirectory
4. Never produce unbounded output — always pipe through `head`/`grep`/`tail`
5. NEVER `Relmem op=index` with broad paths — index already built, use `op=symbols`

### Bash tool timeout

`cc_bash_ipc.ailang`: `DEFAULT_TIMEOUT = 30` seconds. `timeout_secs=0` from the model maps to 30s, capped at 55s (so IPCDispatch's 60s fence always fires last). Already implemented.

### IPCDispatch

60-second hard timeout on all tool calls via `Socket.SetRecvTimeout(fd, 60000)`. After timeout: returns `"tool TIMED OUT (60s): <name>"` to model. `IPCDispatch_Reconnect` called to flush stale socket state.

