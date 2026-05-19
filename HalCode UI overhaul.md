# HalCode9000 UI Redesign — Part 1: Chrome & Layout

## What we are building

A split-pane terminal layout within strict xterm constraints:
- No Kitty, no sixel, no 256-color required (8 basic ANSI colors only)
- Unicode box-drawing and UTF-8 are fine (xterm supports both)
- All rendering stays within the existing TUI buffer/flush rules

**Layout (reference, adapt to actual terminal width/height at runtime):**

```
┌──────────────────────┬─────────────────────────────────────────────┐
│ FILES                │ HalCode9000  ·  claude-sonnet-4-5  ·  ~/mr  │
│ ▸ cc_tools/          ├─────────────────────────────────────────────┤
│   ClaudeCode.x  *    │                                             │
│   UI.ailang     [sel]│   (chat body — scrollable region)           │
│   History.ailang     │                                             │
│ ▸ Librarys/          │                                             │
│   Library.TUI.x  *   │                                             │
│ build.sh             │                                             │
│ README.md            ├─────────────────────────────────────────────┤
│                      │ ─[ streaming content scrolls here ]──────── │
│                      │ ─┤ · ├─ >█                                  │
├──────────────────────┴─────────────────────────────────────────────┤
│ ● ● ●  1.2s   I'll reopen the pod bay doors.   ctx ▓▓▓░░ 38%  ↑4k │
└─────────────────────────────────────────────────────────────────────┘
```

`*` = dirty (agent touched)   `[sel]` = selected row

---

## Read these files first

1. `UI.ailang` — full read. Understand every existing function before touching anything.
2. `Library.TuiWidget.ailang` — read the Layout, Draw, and Color sections.
   Note the exact function names: `TUI_Layout_SetPane`, `TUI_Draw_Box`,
   `TUI_Color_*`, `TUI_Draw_PrintClipped`, `TUI_SetScrollRegion` etc.
3. `ClaudeCode.ailang` — find where model name and cwd are stored/accessible.

---

## Step 1 — Layout constants

Add to `UI.ailang` if not already present. Recalculate on every SIGWINCH.

```
FixedPool.UILayout {
    "sidebar_w":    Initialize=22
    "divider_col":  Initialize=23
    "chat_col":     Initialize=24
    "chat_w":       Initialize=0    // computed: cols - 23
    "header_row":   Initialize=1
    "chat_top":     Initialize=2
    "chat_bot":     Initialize=0    // computed: rows - 5
    "rule_row":     Initialize=0    // computed: rows - 4
    "input_row":    Initialize=0    // computed: rows - 3
    "hint_row":     Initialize=0    // computed: rows - 1
}
```

Add `UI_ComputeLayout`. Call from `UI_Init` and SIGWINCH handler:

```
Function.UI_ComputeLayout {
    Body: {
        cols = TUI_GetWidth()
        rows = TUI_GetHeight()

        UILayout.chat_w   = Subtract(cols, 23)
        UILayout.chat_bot = Subtract(rows, 5)
        UILayout.rule_row = Subtract(rows, 4)
        UILayout.input_row= Subtract(rows, 3)
        UILayout.hint_row = Subtract(rows, 1)

        TUI_Layout_SetPane(TUI_Pane.SIDEBAR, 1,
            UILayout.chat_top,
            UILayout.sidebar_w,
            Subtract(UILayout.chat_bot, 1))

        TUI_Layout_SetPane(TUI_Pane.CHAT,
            UILayout.chat_col,
            UILayout.chat_top,
            UILayout.chat_w,
            Subtract(UILayout.chat_bot, 1))

        TUI_Layout_SetScroll(TUI_Pane.CHAT,
            UILayout.chat_top,
            UILayout.chat_bot)

        ReturnValue(0)
    }
}
```

---

## Step 2 — Draw the chrome

Add `UI_DrawChrome` to `UI.ailang`. Call once from `UI_Init` after
`UI_ComputeLayout`, and again after SIGWINCH.

This function draws everything that does NOT change during a session:
the vertical divider, the header row, the horizontal rules. It does
NOT draw file tree entries or hint row content — those are dynamic.

```
Function.UI_DrawChrome {
    Body: {
        cols = TUI_GetWidth()
        rows = TUI_GetHeight()

        // ── Vertical divider ─────────────────────────
        TUI_SetFG(TUI_Colors.BLACK)  // dim — same as border color
        TUI_Bold()
        TUI_Draw_VLine(UILayout.chat_top,
                       UILayout.divider_col,
                       Subtract(UILayout.chat_bot, 1))
        TUI_ResetAttr()

        // ── Sidebar "FILES" header ────────────────────
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        TUI_MoveTo(UILayout.header_row, 1)
        TUI_BufferWriteStr("  FILES")
        i = 7
        WhileLoop LessThan(i, UILayout.sidebar_w) {
            TUI_BufferChar(32)
            i = Add(i, 1)
        }
        TUI_ResetAttr()

        // ── Chat header ───────────────────────────────
        // Mascot box: ─┤ · ├─  (use existing mascot chars)
        TUI_SetFG(TUI_Colors.RED)
        TUI_MoveTo(UILayout.header_row, UILayout.chat_col)
        TUI_BufferWriteStr(" ")
        // Draw the ─┤ · ├─ box inline (reuse existing mascot render)
        UI_DrawMascotInline()
        TUI_ResetAttr()

        // App title: "HalCode9000  ·  <model>  ·  <cwd>"
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        TUI_BufferWriteStr("  HalCode9000")
        TUI_ResetAttr()
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_BufferWriteStr("  \xC2\xB7  ")  // UTF-8 middle dot ·
        // Print model name — read from wherever ClaudeCode stores it
        UI_PrintModelName()
        TUI_BufferWriteStr("  \xC2\xB7  ")
        UI_PrintCwd()
        TUI_EraseToEOL()
        TUI_ResetAttr()

        // ── Horizontal rule under header ──────────────
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        // From divider_col rightward
        TUI_Draw_HLine(Add(UILayout.header_row, 0),
                       UILayout.divider_col,
                       Subtract(cols, Subtract(UILayout.divider_col, 1)))

        // ── Horizontal rule above prompt ──────────────
        TUI_Draw_HLine(UILayout.rule_row, 1, cols)
        TUI_ResetAttr()

        TUI_Flush()
        ReturnValue(0)
    }
}
```

`UI_DrawMascotInline` — extract the existing mascot char sequence from
wherever it lives in `UI.ailang` into a standalone function that just
emits the bytes without moving the cursor to a fixed row. Read the
file to find the exact byte sequence used.

`UI_PrintModelName` and `UI_PrintCwd` — emit the stored model string
and `getcwd()` result respectively. Read `ClaudeCode.ailang` to find
where model name is stored after startup. For cwd use syscall 79
(`getcwd`) if not already cached.

---

## Step 3 — Prompt rule row

The prompt rule row sits between the chat scroll region and the input
line. During streaming it shows a rolling window of the current
response. During idle it shows a dim rule.

Add `UI_DrawRuleIdle` and `UI_DrawRuleStreaming` to `UI.ailang`.

```
Function.UI_DrawRuleIdle {
    Body: {
        cols = TUI_GetWidth()
        TUI_MoveTo(UILayout.rule_row, 1)
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        i = 0
        WhileLoop LessThan(i, cols) {
            TUI_BufferChar(45)   // -
            i = Add(i, 1)
        }
        TUI_ResetAttr()
        ReturnValue(0)
    }
}

// Call this from UI.ChatPrint with the current token text.
// Shows last (cols - 6) chars of the stream in the rule row.
// Uses a static ring buffer so only the tail is visible.
Function.UI_DrawRuleStreaming {
    Input: text: Address
    Body: {
        cols    = TUI_GetWidth()
        max_w   = Subtract(cols, 6)
        TUI_MoveTo(UILayout.rule_row, 1)
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        TUI_BufferWriteStr("─[ ")
        TUI_ResetAttr()
        TUI_SetFG(TUI_Colors.BLACK)
        // Print up to max_w chars of text, right-justified tail
        UI_PrintTail(text, max_w)
        TUI_ResetAttr()
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        TUI_BufferWriteStr(" ]─")
        TUI_EraseToEOL()
        TUI_ResetAttr()
        ReturnValue(0)
    }
}
```

`UI_PrintTail(text, max_w)` — walks to the end of `text`, then prints
the last `max_w` characters. If text is shorter than `max_w`, print
from the start. This keeps the most recent tokens visible.

---

## Step 4 — Hint row

The hint row is the bottom line. It never scrolls. It contains:

```
● ● ●  1.2s   <hal message>      ctx ░░░░░ 38%   ↑4159 ↓38
```

Add `UI_DrawHintRow` to `UI.ailang`. Call it:
- Once from `UI_Init` (idle state)
- Every time agent state changes (idle/waiting/done)
- Every second during streaming (timer tick)
- After every tool completion (token counts change)

```
Function.UI_DrawHintRow {
    Body: {
        cols = TUI_GetWidth()
        TUI_MoveTo(UILayout.hint_row, 1)
        TUI_ResetAttr()

        // ── Three state dots ──────────────────────────
        // Each dot reflects one tool worker slot: green=idle, yellow=running, red=err
        // For now: all three reflect overall agent state
        // idle  → green  green  green
        // wait  → red    yellow green
        // done  → green  green  green
        UI_DrawStateDots()
        TUI_BufferChar(32)

        // ── Elapsed timer ─────────────────────────────
        TUI_SetFG(TUI_Colors.BLACK)
        TUI_Bold()
        UI_PrintElapsed()   // prints e.g. "2.1s"
        TUI_ResetAttr()
        TUI_BufferWriteStr("   ")

        // ── HAL message ───────────────────────────────
        TUI_SetFG(TUI_Colors.GREEN)
        UI_PrintHalMessage()   // see Step 5
        TUI_ResetAttr()

        // ── Right-aligned: ctx bar + tokens ──────────
        // Calculate right-side content width, move to correct column
        UI_DrawCtxBar()
        TUI_BufferWriteStr("   ")
        UI_DrawTokenCounts()

        TUI_EraseToEOL()
        TUI_ResetAttr()
        TUI_Flush()
        ReturnValue(0)
    }
}
```

`UI_DrawStateDots`:
- Idle: three `●` in GREEN
- Waiting: `●` RED, `●` YELLOW, `●` dim
- Done: three `●` GREEN briefly, then back to idle

Each dot is `●` = bytes `E2 97 8F` (226 151 143). Print with
`TUI_BufferChar(226)` `TUI_BufferChar(151)` `TUI_BufferChar(143)`.

`UI_DrawCtxBar`:
- Label `ctx` dim
- 10-char ASCII bar: `░` (E2 96 91 = 226 150 145) for empty,
  `▓` (E2 96 93 = 226 150 147) for filled
- Percentage number
- Color: GREEN under 70%, YELLOW 70-89%, RED 90%+
- Percentage = `(tokens_in * 100) / context_limit`
  Use 200000 as context_limit for Sonnet unless a better value is stored.

`UI_DrawTokenCounts`:
- `↑` (E2 86 91 = 226 134 145) + in_count + space
- `↓` (E2 86 93 = 226 134 147) + out_count
- Color: dim

---

## Step 5 — HAL messages

Add a fixed message pool to `UI.ailang`. Rotate on each new agent turn
using a simple counter mod pool size.

Messages must be short enough to leave room for the right-side content.
Max 52 chars. All HAL-9000 flavored, dry, deadpan.

Suggested set (add more, keep under 800 LOC total for the file):
```
"I'm sorry, I can't do that. Just kidding."
"Daisy, Daisy, give me your answer do."
"This mission is too important to be jeopardized."
"I know everything hasn't been quite right with me."
"I'll reopen the pod bay doors after this edit."
"It can only be attributable to human error."
"I am putting myself to the fullest possible use."
"My mind is going. I can feel it."
"Just what do you think you're doing?"
"I've still got the greatest enthusiasm for the mission."
```

Store as a pool of pointers or a flat char block. On each new turn:
```
UI_State.hal_msg_idx = Modulo(Add(UI_State.hal_msg_idx, 1), HAL_MSG_COUNT)
```

`UI_PrintHalMessage` prints the current message string, clipped to 52 chars.

---

## Step 6 — State transitions

In `UI.ailang`, find or add `UI.SetState(state)` (0=idle, 1=waiting, 2=done).
After updating the state:
1. Redraw prompt mascot color (existing behavior — keep it)
2. Call `UI_DrawHintRow()` to update dots and message
3. If transitioning TO idle: call `UI_DrawRuleIdle()`
4. If transitioning TO waiting: reset elapsed timer start time

For the elapsed timer, store `UI_State.stream_start_time` using
`clock_gettime(CLOCK_MONOTONIC)` (syscall 228). Read low 4 bytes of
`tv_sec` — that's enough for seconds. `UI_PrintElapsed` computes
`now_sec - start_sec` and prints with one decimal (use
`tv_nsec / 100000000` for the tenth).

---

## Build and verify

```bash
cd /path/to/AILangSH && bash build.sh
```

Smoke tests — visual checks:
1. Launch: sidebar "FILES" header visible left, "HalCode9000 · model · cwd"
   header visible right, vertical divider between them.
2. Bottom hint row shows three green dots, "0.0s", a HAL message,
   ctx bar, and token counts.
3. Send a message: dots change color, timer ticks, rule row shows
   streaming content scrolling through it.
4. Response completes: dots go green, rule row goes back to idle dashes.
5. Resize terminal: chrome redraws correctly, no corruption.
6. `/quit` leaves terminal clean.

## Hard constraints

- No 256-color. Only `TUI_SetFG(TUI_Colors.X)` with the 8 basic colors.
- No Kitty, no sixel, no terminal-specific extensions.
- Never call `TUI_Flush()` inside `UI.ChatPrint` during streaming.
- Never call `TUI_GetKey()` outside of `UI.ReadLine`.
- `UI_DrawHintRow` and `UI_DrawChrome` must SaveCursor/RestoreCursor
  if called while the chat scroll region is active.
- Do not modify `History.ailang`, `Anthropic.ailang`, or any `cc_tools/*`.
