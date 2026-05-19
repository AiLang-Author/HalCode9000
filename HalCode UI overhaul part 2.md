# HalCode9000 UI Redesign — Part 2: Live Data

## Prerequisites

Part 1 must be complete and building cleanly before starting this.
The chrome shell (sidebar, header, hint row, rule row) must be visible.

---

## Read these files first

1. `UI.ailang` — full read of Part 1 changes.
2. `IPCDispatch.ailang` — find where tool calls are dispatched and
   results received. Note exact variable names for tool name, args,
   result content, and ok/err status.
3. `Library.TuiWidget.ailang` — read the Tree and Block sections.
   Note exact function signatures for `TUI_Tree_Add`,
   `TUI_Tree_MarkDirty`, `TUI_Tree_Render`, `TUI_Block_Begin`,
   `TUI_Block_End`, `TUI_Block_AppendOutput`, `TUI_Block_Render`.
4. `Anthropic.ailang` — find where `thinking` type SSE blocks are
   handled (or not). Note where text delta tokens are emitted.

---

## Step 1 — Populate the file tree on startup

In `ClaudeCode.ailang`, after IPC services are launched and before
entering the agent loop, populate the sidebar tree.

Call `TUI_Tree_Clear()` then dispatch a synchronous `LS` tool call
on the current working directory. Parse the result line by line:

- Lines ending in `/` or identified as directories → `TUI_Tree_Add`
  with `FL_DIR` flag set, depth=0
- All other lines → `TUI_Tree_Add` with flags=0, depth=0

Strip whitespace and the trailing `/` from directory names before
storing. Cap at 28 entries (leave room for the header row).

After populating, call:
```
TUI_SaveCursor()
TUI_Tree_Render(TUI_Pane.SIDEBAR)
TUI_Flush()
TUI_RestoreCursor()
```

---

## Step 2 — Mark dirty files from IPCDispatch

In `IPCDispatch.ailang`, after a tool result is received, add dirty
marking for tools that write to disk.

Read the file to find the exact variable holding the tool name string.
Then after the result comes back:

```
// After result received:
IfCondition Or(
    StringEqual(tool_name, "Write"),
    StringEqual(tool_name, "Edit")
) ThenBlock: {
    // Extract path from call args — read how args are stored
    // in IPCDispatch and use the correct variable
    TUI_Tree_MarkDirty(call_path)
    TUI_SaveCursor()
    TUI_Tree_Render(TUI_Pane.SIDEBAR)
    TUI_Flush()
    TUI_RestoreCursor()
}
```

If `StringEqual` does not exist in AILang, implement a byte-by-byte
comparison inline or add `StrEqual(a, b)` to a utility library.
Do not guess — read the existing codebase for how string comparison
is done elsewhere.

---

## Step 3 — Tool blocks

In `IPCDispatch.ailang`, bracket every tool dispatch with block calls.

**Before dispatch:**
```
block_idx = TUI_Block_Begin(tool_name)
```

**After result received:**
```
TUI_Block_End(block_idx, result_ok, 0)
TUI_Block_AppendOutput(block_idx, result_content)
TUI_SaveCursor()
UI_RenderToolBlocks()
TUI_Flush()
TUI_RestoreCursor()
```

Pass `0` for duration — timing can be added later.

Add `UI_RenderToolBlocks` to `UI.ailang`:

```
Function.UI_RenderToolBlocks {
    Body: {
        // Render in the 8 rows immediately above the rule row,
        // in the chat pane column range only.
        start = Subtract(UILayout.rule_row, 9)
        TUI_Block_Render(start, UILayout.chat_w, 8)
        ReturnValue(0)
    }
}
```

**Tool block visual style** (implement inside `TUI_Block_Render` in
`Library.TuiWidget.ailang` if not already matching this):

Each block renders as one header line:

```
<icon> <toolname>  <duration>  [+]
```

- `●` (E2 97 8F) yellow = running
- `✓` (E2 9C 93) green  = ok
- `✗` (E2 9C 97) red    = err

Left border via color only — no box drawing on the left edge.
Background: none. Text colors from basic 8-color set only:
- Running: YELLOW icon, WHITE name
- OK:      GREEN icon, dim name
- Err:     RED icon, dim name
- Duration and `[+]/[-]`: dim (BLACK bold)

When expanded, output lines indent 2 spaces, color dim GREEN.

---

## Step 4 — Thinking hints

In `Anthropic.ailang`, find where SSE events are routed by type.
Look for handling of `content_block_start` with type `thinking`,
or `content_block_delta` with type `thinking_delta`.

If thinking blocks are currently discarded or printed to chat,
intercept them and route to `UI_SetThinkingHint` instead.

Add to `UI.ailang`:

```
FixedPool.UI_Thinking {
    "buf":      Initialize=0
    "len":      Initialize=0
    "BUF_MAX":  Initialize=120
}

Function.UI_Thinking_Init {
    Body: {
        UI_Thinking.buf = Allocate(UI_Thinking.BUF_MAX)
        UI_Thinking.len = 0
        ReturnValue(0)
    }
}

Function.UI_Thinking_Shutdown {
    Body: {
        IfCondition NotEqual(UI_Thinking.buf, 0) ThenBlock: {
            Deallocate(UI_Thinking.buf, UI_Thinking.BUF_MAX)
            UI_Thinking.buf = 0
        }
        ReturnValue(0)
    }
}

// Called from Anthropic.ailang with each thinking delta chunk
Function.UI_SetThinkingHint {
    Input: text: Address
    Body: {
        // Append to buffer, wrapping at BUF_MAX (ring)
        i = 0
        WhileLoop 1 {
            ch = GetByte(text, i)
            IfCondition EqualTo(ch, 0) ThenBlock: { BreakLoop }
            pos = Modulo(UI_Thinking.len, Subtract(UI_Thinking.BUF_MAX, 1))
            SetByte(UI_Thinking.buf, pos, ch)
            UI_Thinking.len = Add(UI_Thinking.len, 1)
            i = Add(i, 1)
        }
        // Null-terminate tail
        tail = Modulo(UI_Thinking.len, Subtract(UI_Thinking.BUF_MAX, 1))
        SetByte(UI_Thinking.buf, tail, 0)
        UI_DrawThinkingHint()
        ReturnValue(0)
    }
}
```

`UI_DrawThinkingHint` — draws one dim line just above the tool blocks
area. Use the last full sentence from the buffer (scan backward for
`.` or `\n`). If none found, use the last 60 chars.

```
Function.UI_DrawThinkingHint {
    Body: {
        hint_row = Subtract(UILayout.rule_row, 10)
        TUI_SaveCursor()
        TUI_MoveTo(hint_row, UILayout.chat_col)
        TUI_SetFG(TUI_Colors.GREEN)
        TUI_BufferWriteStr("\xE2\x86\x92 ")   // → UTF-8
        UI_PrintThinkingTail()                 // last 60 chars of buf
        TUI_EraseToEOL()
        TUI_ResetAttr()
        TUI_Flush()
        TUI_RestoreCursor()
        ReturnValue(0)
    }
}
```

`UI_PrintThinkingTail` — print last min(60, buf_len) chars from
`UI_Thinking.buf`. Walk backward from tail to find start position,
then print forward. Skip newlines (replace with space).

If the model in use does not emit thinking blocks (most non-extended-
thinking models), `UI_SetThinkingHint` will simply never be called
and the hint row stays blank. That is correct behavior — do not
add a fallback that reads assistant text content into it.

---

## Step 5 — Clear state between turns

In `ClaudeCode.ailang`, at the start of each new agent turn (before
sending the API request):

```
TUI_Block_Clear()
UI_Thinking.len = 0
UI_DrawHintRow()
UI_DrawRuleIdle()
```

This resets tool blocks and thinking hint for the new turn so stale
content from the previous turn does not linger.

---

## Step 6 — Chat scroll region activation

In `UI.ailang`, in `UI.ChatPrint` (or equivalent streaming print
function), wrap the print with scroll region activation:

```
TUI_Layout_Activate(TUI_Pane.CHAT)
TUI_Layout_MoveToBottom(TUI_Pane.CHAT)
TUI_Print(text)
TUI_Layout_Deactivate()
```

Also call `UI_DrawRuleStreaming(text)` from this same function to
keep the rule row updated. Do this AFTER deactivating the scroll
region so the rule row write happens in full-screen mode.

**Critical:** do not call `TUI_Flush()` here. The flush boundary
must remain wherever it currently is.

---

## Build and verify

```bash
cd /path/to/AILangSH && bash build.sh
```

Smoke tests — send "list files in this directory":

1. `●` yellow tool block appears for `LS` immediately when dispatch fires.
2. Block changes to `✓` green when result returns.
3. Sidebar shows the cwd files after LS result.
4. Send "write hello to test.txt" — `test.txt` entry in sidebar turns amber.
5. Thinking hint row shows dim green text if using an extended
   thinking model; stays blank otherwise.
6. Rule row shows tokens scrolling during response, returns to
   dashes after.
7. Token counts in hint row update after each turn.
8. HAL message rotates to a new line on each new turn.

## Hard constraints

- 8-color only throughout. No 256-color, no RGB.
- SaveCursor/RestoreCursor around every sidebar or tool block render
  that fires outside the idle input loop.
- Never flush inside ChatPrint during streaming.
- Do not modify `History.ailang` or any `cc_tools/*` file.
- Keep `Library.TuiWidget.ailang` under 800 LOC. If adding to it
  pushes past 800, split the tree and block widgets into separate
  files first.
