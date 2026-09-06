# HalCode 9000 — TUI Cleanup Plan

> **Status:** Architectural inventory + prioritized remediation roadmap
> **Date:** 2025-06-15
> **Source file:** `~/HalCode9000/UI.ailang` (3,165 lines, 99 functions)
> **Supporting libs:** `Library.TUI`, `Library.ANSI`, `Library.ChatLog`, `Library.Arena`, `Library.StringUtils`
> **Main loop:** `~/HalCode9000/HalCode9000.ailang` (SSE read loop with inline `CC_AnimTick` → `UI.AnimTick`)

---

## 1. File Inventory

| File | Location | Role |
|------|----------|------|
| `UI.ailang` | `~/HalCode9000/UI.ailang` | Terminal display surface — ANSI escape rendering, cursor tracking, scroll-back ring buffer, input line editor, mascot animation |
| `HalCode9000.ailang` | `~/HalCode9000/HalCode9000.ailang` | Top-level orchestrator: SSE stream reader → `CC_AnimTick()` → `UI.AnimTick()` → tool dispatch |
| `Library.TUI` | `~/.halcode/lib/Library.TUI` | Raw terminal mode, ANSI output buffering, `TUI_Refresh()` flush, key decode |
| `Library.ANSI` | `~/.halcode/lib/Library.ANSI` | Color escapes, cursor shape, bracketed paste, title strings |
| `Library.ChatLog` | `~/.halcode/lib/Library.ChatLog` | File-backed chat history: append, read-back, scroll-state tracking |
| `Library.Arena` | `~/.halcode/lib/Library.Arena` | Region-based allocation (`Allocate`/`Deallocate`) |
| `Library.StringUtils` | `~/.halcode/lib/Library.StringUtils` | `StringLength`, `StringConcat`, `NumberToString`, `StringToInt` |
| `tui-cleanup.md` | `~/HalCode9000/tui-cleanup.md` | **This document** |

---

## 2. Architecture & Data Flow

```
┌──────────────────────────────────────────────────────┐
│                HalCode9000.ailang                      │
│                                                        │
│  SSE read loop                                         │
│    │                                                   │
│    ├─ HTTP.ReadLineTimeout() — block on curl pipe      │
│    ├─ SSE.FeedLine() — parse event stream              │
│    ├─ Backend.OnEvent() — content_block_delta →        │
│    │     UI.ChatPrint(tok)  ←── appends to chat area   │
│    ├─ CC_AnimTick() ──► UI.AnimTick()                  │
│    │     ├─ UI.PumpInput()   — drain stdin for ESC/int │
│    │     ├─ UI.TickMascot()  — advance frame counter   │
│    │     ├─ UI_PaintBodyRow()— redraw prompt line      │
│    │     ├─ UI_RenderInput() — redraw input buffer     │
│    │     └─ TUI_Refresh()    — flush ANSI buffer       │
│    └─ (end of stream) → UI.SetState(2) → tool dispatch │
└──────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│                     UI.ailang                         │
│                                                        │
│  All state lives in FixedPools (no heap objects):      │
│                                                        │
│  FixedPool.UILayout     — terminal dimensions, chat    │
│    cursor, input buffer, streaming flags, animation    │
│                                                        │
│  FixedPool.UI_ScrollBuf — ring buffer (200-char lines  │
│    × 500 lines) for scroll-back in /scroll mode        │
│                                                        │
│  FixedPool.UIColors     — FG color indices             │
│  FixedPool.UIGlyph      — UTF-8 glyph strings          │
│  FixedPool.UISys        — syscall numbers              │
│                                                        │
│  Rendering is procedural: each paint function emits    │
│  ANSI bytes directly into Library.TUI's buffer via     │
│  TUI_MoveTo / TUI_BufferWriteStr / TUI_BufferChar,     │
│  then TUI_Refresh() flushes to stdout.                 │
│                                                        │
│  There is NO widget tree, NO dirty-region tracking,    │
│  NO front/back screen buffer. The terminal IS the      │
│  display — all state is terminal state.                │
└──────────────────────────────────────────────────────┘
```

### Critical paths

1. **Stream token → screen** — SSE delta byte arrives → Backend calls `UI.ChatPrint` → byte-accumulator in `UILayout.line_acc` → on `\n`, calls `ChatLog_Append(97, line)` → TUI buffer → `TUI_Refresh()` flushes on next tick
2. **Anim tick** — `UI.AnimTick` → `UI.PumpInput` (poll stdin for ESC/Ctrl+C) → `UI.TickMascot` → repaint prompt row → `TUI_Refresh()`. Enforced at 25 FPS (40ms minimum between ticks).
3. **Input editing** — `UI.ReadLine` polls stdin with 15ms timeout, handles backspace/arrows/paste/home/end, rebuilds the input buffer `UILayout.input_buf` byte-by-byte
4. **Scroll-back** — `UI_ScrollBack` / `UI_ScrollForward` seek through the ChatLog file (`ChatLog_State.path`) counting newlines to reposition `ChatLog_State.file_pos`, then `ChatLog_ReadBack(chat_rows)` repaints

### Single-threaded guarantee

Everything runs in one thread. There is no concurrency — no mutexes, no atomics, no signal handlers. The "display loop" and "streaming loop" are the **same loop**: each SSE read iteration calls `CC_AnimTick()` which calls `UI.AnimTick()` which paints one frame. There is no race between streaming and rendering.

---

## 3. Function Catalog (all 99 functions in UI.ailang)

### 3.1 Lifecycle (4 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 1 | `UI.Init` | 196 | Allocate input buffer, init TUI, clear ISIG termios flag, exit alt screen, init ChatLog, draw chrome |
| 2 | `UI.Shutdown` | 274 | BracketedPasteOff, restore cursor shape, TUI_Shutdown |
| 3 | `UI.ClearChat` | 286 | Clear chat region rows, reset cursor to top |
| 4 | `UI.QuerySize` | 3165 | Calls `UI_RecomputeLayout` — public entry for resize |

### 3.2 Layout & Row Geometry (6 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 5 | `UI_RecomputeLayout` | 303 | Read terminal size via `TUI_GetScreenSize()`, recompute `chat_top`/`chat_bot`/`rule_row`/`input_row`/`hint_row` |
| 6 | `UI_BarWidth` | 331 | Clamp `cols` to `max_bar` (512), floor at 10 |
| 7 | `UI_TopRuleRow` | 341 | Returns `chat_bot + 1` |
| 8 | `UI_BodyRow` | 342 | Returns `chat_bot + 2` |
| 9 | `UI_BotRuleRow` | 343 | Returns `chat_bot + prompt_h - 1` |
| 10 | `UI_HintRow` | 344 | Returns `chat_bot + prompt_h` |

### 3.3 Prompt Frame Painting (7 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 11 | `UI_PromptColor` | 350 | Set FG color based on `ui_state` (orange wait / red error / green done) |
| 12 | `UI_PaintTopRule` | 364 | Draw top rule row with state dots, context bar, token counts |
| 13 | `UI_PaintBotRule` | 466 | Draw bottom rule (─ repeated) |
| 14 | `UI_PaintBodyRow` | 485 | Draw mascot + "> " prompt in body row |
| 15 | `UI_PaintHint` | 501 | Draw hint row (current hint text or streaming counters) |
| 16 | `UI_RenderInput` | 643 | Render the input buffer at cursor position with scroll handling |
| 17 | `UI_PaintPromptFrame` | 702 | Paint full prompt frame (rules + body + hint) |

### 3.4 Scroll-Back Ring Buffer (7 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 18 | `UI_ScrollBuf_Init` | 719 | Allocate ring buffer (`max_lines × line_w`), zero-init |
| 19 | `UI_ScrollBuf_FinalizeLine` | 733 | Commit current line to ring slot, advance count |
| 20 | `UI_ScrollBuf_AddChar` | 750 | Add a byte to current line, skip ANSI escape sequences, finalize on `\n` |
| 21 | `UI_ScrollBuf_AddStr` | 777 | Feed a null-terminated string through `AddChar` |
| 22 | `UI_ScrollBuf_Render` | 791 | Paint visible portion of ring buffer into chat region, draw divider VLine |
| 23 | `UI_ScrollBuf_PageUp` | 825 | Increment `view_offset`, clamp, re-render |
| 24 | `UI_ScrollBuf_PageDown` | 840 | Decrement `view_offset`, clamp at 0, re-render |

### 3.5 Chat Output (10 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 25 | `UI.ChatPrint` | 1013 | Append string to chat region: byte-by-byte with soft-wrap at `cols`, newline advances cursor + scrolls |
| 26 | `UI.ChatPrintln` | 1104 | `ChatPrint(s)` then advance cursor + newline |
| 27 | `UI.ChatPrintDim` | 1124 | Print in DIM_FG color |
| 28 | `UI.ChatTag` | 1148 | Print colored tag line (e.g. "● Read(...)") |
| 29 | `UI_AdvanceChatCursor` | 994 | Move cursor down 1 row, scroll if at `chat_bot`. Returns 1 if scrolled (caller repaints prompt) |
| 30 | `UI_ScrollChatUp` | 974 | Scroll chat region up 1 line (uses scroll region if streaming, else full-screen approach) |
| 31 | `UI_DrawTurnSeparator` | 864 | Draw "───" divider before each user turn |
| 32 | `UI_RefreshDivider` | 856 | Redraw VLine divider after terminal scroll clears it |
| 33 | `UI_RepaintPromptFrameOnly` | 1136 | Repaint just the prompt frame (used after scroll disturbs it) |
| 34 | `UI_PrintTail` | 2253 | Print last N lines of chat to stdout (for session log) |

### 3.6 ChatLog File-Backed History (2 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 35 | `UI_ScrollBack` | 891 | Open ChatLog file, seek backward from `file_pos` counting N newlines, reposition, `ChatLog_ReadBack` repaint |
| 36 | `UI_ScrollForward` | 932 | Open ChatLog file, seek forward from `file_pos` counting N newlines, reposition, repaint; snap to bottom at EOF |

### 3.7 Input Line Editor (4 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 37 | `UI.ReadLine` | 1491 | Blocking input loop: poll stdin (15ms timeout), handle keys (arrows, backspace, home, end, paste, Ctrl+C→/quit, Enter→submit), animate mascot during idle |
| 38 | `UI_CalcInputLines` | 1220 | Compute how many visual rows the input buffer needs (accounts for soft-wrap at `cols`) |
| 39 | `UI_AdjustPromptHeight` | 1256 | Grow/shrink `prompt_h` to fit multi-line input, reflow layout, repaint |
| 40 | `UI_GetHintText` | 186 | Return hint string based on current state |

### 3.8 Streaming State Machine (6 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 41 | `UI.SetState` | 1323 | Set `ui_state` (0=idle, 1=streaming, 2=done): manage scroll region, reset timers, repaint |
| 42 | `UI.PumpInput` | 1358 | Non-blocking stdin drain during streaming: check for ESC (interrupt), Ctrl+O (toggle tools), Enter (submit early) |
| 43 | `UI.HasInterrupt` | 1767 | Return `UILayout.interrupted` |
| 44 | `UI.ConsumeInterrupt` | 1773 | Reset interrupt flag, return any partial input text |
| 45 | `UI.AnimTick` | 1812 | 25 FPS tick: `PumpInput` → `TickMascot` → update elapsed timer → draw streaming rule → repaint |
| 46 | `UI.SetReasoning` | 1756 | Set `reasoning_str` for the streaming rule display |

### 3.9 Status Display (6 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 47 | `UI.SetTokens` | 1719 | Store token counts (`tok_in`, `tok_out`) |
| 48 | `UI.SetActiveTool` | 1737 | Set `active_tool` name for status display |
| 49 | `UI.SetQuote` | 1746 | Set the rotating quote string |
| 50 | `UI.SetModelName` | 2095 | Store model name for header display |
| 51 | `UI.SetProjectThread` | 2190 | Store project/thread slugs for display |
| 52 | `UI_DrawStateDots` | 2291 | Draw colored dots in top rule (idle/streaming/done indicator) |

### 3.10 Chrome & Rules (7 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 53 | `UI_DrawChrome` | 2430 | Draw full static chrome (rules, hint, prompt) |
| 54 | `UI_DrawHintRow` | 2367 | Draw the hint row with context info |
| 55 | `UI_DrawRuleIdle` | 2112 | Draw top rule in idle state |
| 56 | `UI_DrawRuleStreaming` | 2222 | Draw top rule during streaming (with reasoning text) |
| 57 | `UI_DrawCtxBar` | 2316 | Draw context pressure bar |
| 58 | `UI_DrawTokenCounts` | 2354 | Draw token in/out counts |
| 59 | `UI_GetCwd` | 2102 | Read CWD for display |

### 3.11 Tool Call Display (4 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 60 | `UI.ToolGroupStart` | 2452 | Begin compact tool group display |
| 61 | `UI.ToolCallStart` | 2471 | Display tool call header (name + args summary) |
| 62 | `UI.ToolCallEnd` | 2548 | Display tool result indicator (✓/✗) |
| 63 | `UI.ToolGroupEnd` | 2588 | End tool group, print summary line |

### 3.12 Mascot Animation (8 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 64 | `UI_DrawMascot` | 1951 | Draw the 1-line mascot `─┤ · ├─` with animated eye |
| 65 | `UI.TickMascot` | 1984 | Advance `mascot_tick`, compute `mascot_frame` |
| 66 | `UI_MascotEyePos` | 1864 | Return eye column offset (always 1 = center) |
| 67 | `UI_SetMascotColor` | 1873 | Set color: pulses during wait, prompt color otherwise |
| 68 | `UI_EmitHalEye` | 1890 | Emit FISHEYE glyph (◎) with animation variants |
| 69 | `UI_EmitLeftArm` | 1911 | Emit left arm with animation variants |
| 70 | `UI_EmitRightArm` | 1931 | Emit right arm with animation variants |
| 71 | `UI.SessionHeader` | 1992 | Print 3-line static mascot + version/model/cwd header |

### 3.13 HAL Messages (2 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 72 | `UI_HALMsgs_Init` | 2064 | Initialize rotating HAL message array |
| 73 | `UI_PrintHalMessage` | 2079 | Print a rotating HAL-9000 themed message |

### 3.14 Help & Boot (3 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 74 | `UI.PrintHelp` | 2617 | Print slash-command help text |
| 75 | `UI.BootScroll` | 2641 | Scroll boot splash text |
| 76 | `UI.StartupAnimation` | 3042 | Full startup animation sequence |

### 3.15 Splash / Boot Animation (5 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 77 | `UI_SplashWrite` | 2682 | Write boot splash line |
| 78 | `UI_SplashSleepMs` | 2692 | Sleep during boot animation |
| 79 | `UI_SplashReadAll` | 2708 | Read entire splash file |
| 80 | `UI_SplashLineWidth` | 2732 | Measure visual width of splash line |
| 81 | `UI.SplashFile` | 2767 | Load and display splash file |

### 3.16 Splash Animation Helpers (5 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 82 | `UI_SplashGetEnv` | 2833 | Read env var for splash |
| 83 | `UIAni_Int` | 2903 | Integer helper for animation |
| 84 | `UIAni_Goto` | 2932 | Cursor positioning during animation |
| 85 | `UIAni_FgRGB` | 2950 | RGB color during animation |
| 86 | `UIAni_LogoRow` | 2977 | Logo row helper |
| 87 | `UIAni_DrawLogoRow` | 2991 | Draw one row of animated logo |

### 3.17 TrueColor Helpers (2 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 88 | `UI_TUI_WriteInt` | 140 | Write integer as decimal digits into TUI buffer |
| 89 | `UI_TUI_FgRGB` | 162 | Emit 24-bit RGB foreground SGR sequence |

### 3.18 Help (1 function)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 90 | `UI.PrintHelp` | 2617 | Print slash-command reference |

### 3.19 Stubs & Aliases (9 functions)

| # | Function | Line | Purpose |
|---|----------|------|---------|
| 91 | `UI_RenderSidebar` | 887 | Stub — returns 0 |
| 92 | `UI_MarkFileRead` | 888 | Stub — returns 0 |
| 93 | `UI_Write` | 3158 | Alias for `UI.ChatPrint` |
| 94 | `UI_Writeln` | 3159 | Alias for `UI.ChatPrintln` |
| 95 | `UI.WriteColor` | 3160 | Alias for `UI.ChatPrint` (ignores color param) |
| 96 | `UI.BootBanner` | 3161 | Stub — empty body |
| 97 | `UI.SplashBoot` | 3162 | Stub — empty body |
| 98 | `UI.RenderPromptArea` | 3163 | Stub — empty body |
| 99 | `UI.RenderPromptClose` | 3164 | Stub — empty body |

---

## 4. Real Architecture Notes

### No widget tree
UI.ailang does not use `Library.TuiWidget`. There is no widget struct, no tree, no dirty tracking. Rendering is **procedural** — each function emits ANSI bytes directly into `Library.TUI`'s output buffer. The terminal's own display is the only "screen buffer."

### No dual-buffer diff
There is no front/back screen cell grid. The original document's Section 3.5 ("Screen Differ & Flush") described imaginary functions. In reality, `TUI_Refresh()` writes `Library.TUI`'s internal buffer to stdout. Terminal scroll regions (`ESC [ r`) are used to constrain scrolling to the chat area during streaming.

### Single-threaded, no signals
- There is NO SIGWINCH handler. Terminal resize must be triggered by calling `UI.QuerySize()` (which calls `UI_RecomputeLayout()`). Currently only called from `UI.Init()` — resize during a session is not automatically detected.
- Ctrl+C is not a signal. `UI.Init` clears the ISIG termios flag so Ctrl+C arrives as byte `0x03` on stdin. `UI.ReadLine` and `UI.PumpInput` check for byte 3 and treat it as `/quit` or interrupt.
- There are NO mutexes, NO atomics, NO volatile sig_atomic_t. Everything is single-threaded.

### Memory model: arena allocation
- `Allocate(N)` — allocate N bytes from the current arena
- `Deallocate(ptr, N)` — release N bytes back (must match allocation size)
- `SetByte(ptr, idx, val)` / `GetByte(ptr, idx)` — byte access
- `StoreValue(ptr, offset, type)` — typed write (e.g. `"dword"`)
- `MemorySet(ptr, val, N)` — fill N bytes with val
- FixedPools hold module-level state — they are the AILang equivalent of global variables

### ChatLog is file-backed
- `ChatLog_Append(fd, data)` writes to a file at `ChatLog_State.path`
- `ChatLog_ReadBack(chat_rows)` reads the last N lines from the file
- `ChatLog_State.file_pos` tracks the read position for scroll-back
- `ChatLog_State.at_bottom` is 1 when viewing live (latest content)

---

## 5. Known Problem Areas

### 🔴 P0 — Critical (will crash or corrupt display)

**P0-1: `UI.ChatPrint` line accumulator overflow (line ~1013)**
- `UILayout.line_acc` is a fixed 512-byte allocation. `UILayout.line_acc_pos` is checked against 510 before each `SetByte`, so bytes beyond position 509 are silently dropped.
- A single line longer than 510 bytes (e.g. a long code block on one line) will be truncated before it reaches `ChatLog_Append`.
- **Fix:** Check for overflow and flush partial content to ChatLog before continuing, or increase the accumulator size.

**P0-2: No 0-guard on `s` parameter in `UI.ChatPrint` (line ~1021)**
- Already guarded: `IfCondition EqualTo(s, 0) ThenBlock: { ReturnValue(0) }` — this is already handled. No crash risk here.

**P0-3: `UI_ScrollBuf_FinalizeLine` writes beyond buffer for lines > `line_w` (line ~733)**
- `UI_ScrollBuf_AddChar` checks `cur_len < line_w - 1` before adding, so overflow is prevented. But if `cur_len` somehow equals `line_w`, the `SetByte` at line 735 writes one byte past the allocation.
- This can't happen via `AddChar` (guarded), but could if `cur_len` is corrupted or set externally.
- **Fix:** Add a bounds check: `IfCondition GreaterEqual(UI_ScrollBuf.cur_len, UI_ScrollBuf.line_w) ThenBlock: { UI_ScrollBuf.cur_len = Subtract(UI_ScrollBuf.line_w, 1) }`

### 🟠 P1 — High (visual glitches, incorrect display)

**P1-4: Terminal resize not detected during session**
- `UI_RecomputeLayout()` is only called from `UI.Init()` and `UI.QuerySize()`. There is no periodic polling of terminal dimensions.
- If the user resizes the terminal mid-session, `UILayout.rows`/`UILayout.cols` become stale. The chat region may not fill the new terminal size, and soft-wrap calculations will be wrong.
- **Fix:** Add a `SystemCall(UISys.SYS_IOCTL, 0, TIOCGWINSZ, ...)` check in `UI.AnimTick` (every ~40ms) or expose `UI.QuerySize` to the main loop.

**P1-5: `UI_ScrollChatUp` during idle uses full-screen scroll approach (line ~974)**
- When `streaming == 0`, the function emits `ESC [J` (clear from cursor to end) then `ESC [999;1H` then newline. This relies on the terminal scrolling the entire screen, which can shift the prompt frame.
- The caller `UI_AdvanceChatCursor` returns 1 when a scroll happened, triggering `UI_RepaintPromptFrameOnly()`, but there's a window where the prompt frame is visually displaced.
- **Fix:** Always use scroll regions (`ESC [top;bot r`) during idle too, so scrolling never affects the prompt area.

**P1-6: `UI.PumpInput` allocates/deallocates 8 bytes per poll iteration (line ~1358)**
- Inside the `WhileLoop 1`, every iteration calls `Allocate(8)` for the pollfd struct, then `Deallocate(pfd, 8)` after the `SystemCall(SYS_POLL, ...)`.
- During streaming with rapid token arrival (~60 tok/s), this creates allocation churn in the arena.
- **Fix:** Hoist the pollfd allocation outside the loop, or use a FixedPool slot for the pollfd bytes.

### 🟡 P2 — Medium (performance, edge cases)

**P2-7: ChatLog file I/O on every message**
- `ChatLog_Append` writes to a file on every completed line. Under rapid streaming, this means many small `write` syscalls.
- `UI_ScrollBack` / `UI_ScrollForward` open and close the ChatLog file on every PageUp/PageDown press.
- **Fix:** Buffer writes in the ChatLog library; keep the file descriptor open during streaming rather than re-opening.

**P2-8: Scroll-back ring buffer has fixed capacity (200 chars × 500 lines)**
- `UI_ScrollBuf.max_lines = 500`. When more than 500 lines are accumulated, old lines are silently overwritten via modulo wrapping.
- The `/scroll` mode only sees the last 500 lines of output.
- **Fix:** Increase `max_lines` or switch to the file-backed ChatLog for scroll-back (which is already used by PageUp/PageDown in the main chat view).

**P2-9: `UI.ChatPrint` byte-at-a-time TUI buffer writes (line ~1013)**
- For each printable byte, `UI.ChatPrint` allocates a 2-byte temp string (`Allocate(2)`), sets byte 0 to the char, byte 1 to 0, calls `TUI_BufferWriteStr(tmp)`, then deallocates.
- This is O(n) allocate/deallocate pairs per output character.
- **Fix:** Accumulate printable characters and flush in chunks, or add a `TUI_BufferChar` call path that doesn't require allocation.

**P2-10: `UI_CalcInputLines` cache may be stale (line ~1220)**
- The cache keys are `input_len_cached`, `input_cols_cached`, `input_lines_cached`. If `cols` changes (resize) but `RecomputeLayout` isn't called, the cache won't invalidate and soft-wrap will be wrong.
- **Fix:** Include `UILayout.cols` as part of the cache key check.

---

## 6. Cleanup Plan (Prioritized)

### Phase 1 — Stabilize (crash prevention)
| Step | Item | Effort | Risk |
|------|------|--------|------|
| 1.1 | P0-1: Handle line_acc overflow with partial flush | 15 min | None |
| 1.2 | P0-3: Add bounds guard in ScrollBuf_FinalizeLine | 5 min | None |
| 1.3 | Test: long single-line input, fresh start, quit | 10 min | — |

### Phase 2 — Correctness (visual glitches)
| Step | Item | Effort | Risk |
|------|------|--------|------|
| 2.1 | P1-4: Periodic terminal size poll in AnimTick | 20 min | Low |
| 2.2 | P1-5: Use scroll regions in idle mode too | 15 min | Low |
| 2.3 | P2-10: Fix CalcInputLines cache invalidation | 10 min | None |
| 2.4 | Test: resize during streaming, idle, and input | 15 min | — |

### Phase 3 — Performance polish
| Step | Item | Effort | Risk |
|------|------|--------|------|
| 3.1 | P1-6: Hoist pollfd allocation out of PumpInput loop | 10 min | None |
| 3.2 | P2-9: Accumulate bytes before TUI buffer writes | 20 min | Low |
| 3.3 | P2-7: Keep ChatLog fd open during streaming | 15 min | Low |
| 3.4 | Profile: 10K message stress test, 120×40 terminal | 30 min | — |

### Phase 4 — Deferred (nice to have)
| Step | Item |
|------|------|
| 4.1 | Replace `UI_ScrollBuf` ring buffer with ChatLog file-backed reads (unify scroll paths) |
| 4.2 | Fill in stubs: `UI_RenderSidebar`, `UI_MarkFileRead` with file-tree integration |
| 4.3 | Mouse event support (SGR tracking) |
| 4.4 | Split UI.ailang into sub-modules (paint, input, scroll, mascot, splash) |

---

## 7. Key FixedPool Constants & Limits

| Pool.Key | Value | Purpose |
|----------|-------|---------|
| `UILayout.input_cap` | 65536 | Input buffer max bytes |
| `UILayout.max_bar` | 512 | Maximum bar/rule width |
| `UILayout.prompt_h` | 4 (default) | Prompt frame height in rows |
| `UI_ScrollBuf.line_w` | 200 | Max chars per scroll-back line |
| `UI_ScrollBuf.max_lines` | 500 | Ring buffer capacity in lines |
| `UILayout.line_acc` | 512 bytes | ChatPrint line accumulator |
| `UILayout.tool_names_buf` | 512 bytes | Tool group name buffer |
| `UILayout.proj_slug` | 64 bytes | Project slug buffer |
| `UILayout.thread_slug` | 64 bytes | Thread slug buffer |
| `UILayout.mascot_tick` / `mascot_frame` | — | Animation frame counters (25 FPS, 10-frame eye cycle) |
| `UISys.SYS_POLL` | 7 | poll(2) syscall number |
| `UISys.SYS_IOCTL` | 16 | ioctl(2) syscall number |

---

## 8. Dependencies (LibraryImport order)

```
Library.Arena          (no deps — system allocator)
Library.StringUtils    → Library.Arena
Library.JSON           → Library.Arena, Library.StringUtils
Library.ANSI           (no deps)
Library.TUI            → Library.ANSI
Library.ChatLog        → Library.TUI, Library.Arena
UI.ailang              → Library.TUI, Library.ANSI, Library.ChatLog, Library.Arena, Library.StringUtils, Library.JSON
HalCode9000.ailang     → UI, History, IPCDispatch, Backend, Auth, HTTP, SSE
```

---

## 9. Working State

- **Terminal:** 24-bit color capable (kitty), typically 120×40
- **Source:** `~/HalCode9000/UI.ailang` (125,872 bytes, 3,165 lines, 99 functions)
- **Build:** `./ailang.x Applications/HalCode9000/UI.ailang ui.o` (compiled to object, linked into HalCode9000.x)
- **Real functions:** 90 implemented + 9 stubs/aliases
- **No test harness:** All testing is manual via running HalCode9000.x
