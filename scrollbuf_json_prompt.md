# HalCode9000 — NDJSON Chat Buffer & Scroll Back

Replace the in-memory scroll buffer with a newline-delimited JSON
file. Every chat line is appended to `~/.halcode/chat_current.ndjson`
as it is printed. Scroll back reads backward from that file.
No ring buffer, no memory cap, survives crashes, inspectable from
outside the process.

---

## Read these files first

1. `UI.ailang` — full read. Find every place text is printed to
   the chat area. These all become `UI_Chat_AppendLine` calls.
2. `Library.JSON.ailang` — check what JSON building primitives
   exist. We need to emit a single-line JSON object per chat line.
3. `ClaudeCode.ailang` — find session start and `/clear` handling.

---

## Step 1 — Library.ChatLog.ailang

Create `Librarys/Library.ChatLog.ailang`. Keep under 300 LOC.

```ailang
// ============================================
// Library.ChatLog.ailang
// NDJSON append log for chat scroll back
// One JSON object per line:
// {"r":"a","t":"text content","ts":1234567890}
// r = role: "u"=user "a"=assistant "t"=tool "s"=system
// ============================================

FixedPool.ChatLog_Config {
    "PATH_MAX":   Initialize=256
    "LINE_MAX":   Initialize=512
    "READ_BUF":   Initialize=4096
    "KEEP_FILES": Initialize=5
}

FixedPool.ChatLog_State {
    "fd":         Initialize=-1
    "path":       Initialize=0
    "read_buf":   Initialize=0
    "line_buf":   Initialize=0
    "file_pos":   Initialize=0   // current read position for scroll
}
```

### ChatLog_Init

Allocate `path`, `read_buf`, `line_buf`. Build the path
`~/.halcode/chat_current.ndjson` using `getenv("HOME")` or
`/proc/self/environ` scan. Store in `ChatLog_State.path`.

Open the file with `O_WRONLY | O_CREAT | O_APPEND`, mode 0600.
Syscall 2 = open. Flags: O_WRONLY=1, O_CREAT=64, O_APPEND=1024,
combined = 1089. Store fd in `ChatLog_State.fd`.

Also read current file size using `lseek(fd, 0, SEEK_END)`
(syscall 8, whence=2) and store in `ChatLog_State.file_pos` as
the initial scroll anchor (bottom of file = latest content).

```ailang
Function.ChatLog_Init {
    Body: {
        ChatLog_State.path     = Allocate(ChatLog_Config.PATH_MAX)
        ChatLog_State.read_buf = Allocate(ChatLog_Config.READ_BUF)
        ChatLog_State.line_buf = Allocate(ChatLog_Config.LINE_MAX)

        // Build path: HOME + "/.halcode/chat_current.ndjson"
        home = // getenv or /proc scan — read existing codebase for
               // how HOME is resolved elsewhere, use same pattern
        ChatLog_BuildPath(home)

        // Open for append-write
        ChatLog_State.fd = SystemCall(2, ChatLog_State.path, 1089, 420)

        // Seek to end to get file size as initial scroll anchor
        size = SystemCall(8, ChatLog_State.fd, 0, 2)
        ChatLog_State.file_pos = size

        ReturnValue(0)
    }
}
```

### ChatLog_Shutdown

Close the fd. Free allocations.

### ChatLog_Append

Appends one NDJSON line. Builds the object and writes it.

```ailang
Function.ChatLog_Append {
    Input: role: Integer    // 117='u' 97='a' 116='t' 115='s'
    Input: text: Address
    Body: {
        IfCondition EqualTo(ChatLog_State.fd, -1) ThenBlock: {
            ReturnValue(-1)
        }

        buf  = ChatLog_State.line_buf
        pos  = 0

        // {"r":"X","t":"
        ChatLog_BufStr(buf, "{\"r\":\"")  // pos advances
        SetByte(buf, 6, role)
        pos = 7
        ChatLog_BufStr2(buf, pos, "\",\"t\":\"")
        pos = Add(pos, 7)

        // Copy text, escaping " and \ and control chars
        i = 0
        WhileLoop 1 {
            ch = GetByte(text, i)
            IfCondition EqualTo(ch, 0) ThenBlock: { BreakLoop }
            IfCondition GreaterEqual(pos, Subtract(ChatLog_Config.LINE_MAX, 20)) ThenBlock: {
                BreakLoop
            }
            // Escape backslash
            IfCondition EqualTo(ch, 92) ThenBlock: {
                SetByte(buf, pos, 92) pos = Add(pos, 1)
                SetByte(buf, pos, 92) pos = Add(pos, 1)
            } ElseBlock: {
                // Escape double-quote
                IfCondition EqualTo(ch, 34) ThenBlock: {
                    SetByte(buf, pos, 92) pos = Add(pos, 1)
                    SetByte(buf, pos, 34) pos = Add(pos, 1)
                } ElseBlock: {
                    // Replace control chars with space
                    IfCondition LessThan(ch, 32) ThenBlock: {
                        SetByte(buf, pos, 32) pos = Add(pos, 1)
                    } ElseBlock: {
                        SetByte(buf, pos, ch) pos = Add(pos, 1)
                    }
                }
            }
            i = Add(i, 1)
        }

        // Get timestamp via clock_gettime CLOCK_REALTIME (syscall 228, clk=0)
        ts_buf = Allocate(16)
        SystemCall(228, 0, ts_buf)
        ts_sec_lo = GetByte(ts_buf, 0)
        ts_sec_hi = GetByte(ts_buf, 1)
        ts_sec = Add(ts_sec_lo, Multiply(ts_sec_hi, 256))
        Deallocate(ts_buf, 16)

        // "}\n  — close object
        // Write: ","ts":NNNN}\n
        SetByte(buf, pos, 34)  pos = Add(pos, 1)  // "
        SetByte(buf, pos, 44)  pos = Add(pos, 1)  // ,
        // append "ts":NNNN
        ChatLog_BufStr2(buf, pos, "\"ts\":")
        pos = Add(pos, 5)
        pos = ChatLog_BufNum(buf, pos, ts_sec)
        SetByte(buf, pos, 125) pos = Add(pos, 1)  // }
        SetByte(buf, pos, 10)  pos = Add(pos, 1)  // \n
        SetByte(buf, pos, 0)

        // Write to file
        SystemCall(1, ChatLog_State.fd, buf, pos)

        ReturnValue(0)
    }
}
```

Implement `ChatLog_BufStr`, `ChatLog_BufStr2`, `ChatLog_BufNum`
as simple inline helpers that copy a string or number into `buf`
at a given offset and return the new offset.

### ChatLog_ReadBack

Reads N lines backward from `file_pos`. Opens a second fd
read-only for scroll reads so the append fd is never disturbed.

```ailang
Function.ChatLog_ReadBack {
    Input: n_lines: Integer          // how many lines to read back
    Output: Integer                  // actual lines read
    Body: {
        // Open file read-only
        rfd = SystemCall(2, ChatLog_State.path, 0, 0)
        IfCondition LessThan(rfd, 0) ThenBlock: { ReturnValue(0) }

        // We'll collect up to n_lines line-start offsets
        // by scanning backward from file_pos in READ_BUF chunks
        offsets  = Allocate(Multiply(n_lines, 8))  // 8 bytes per offset
        found    = 0
        scan_pos = ChatLog_State.file_pos

        WhileLoop And(GreaterThan(scan_pos, 0), LessThan(found, n_lines)) {
            chunk = ChatLog_Config.READ_BUF
            IfCondition LessThan(scan_pos, chunk) ThenBlock: {
                chunk = scan_pos
            }
            scan_pos = Subtract(scan_pos, chunk)

            // Seek and read chunk
            SystemCall(8, rfd, scan_pos, 0)
            bytes = SystemCall(0, rfd, ChatLog_State.read_buf, chunk)

            // Scan backward through chunk for newlines
            i = Subtract(bytes, 1)
            WhileLoop And(GreaterEqual(i, 0), LessThan(found, n_lines)) {
                ch = GetByte(ChatLog_State.read_buf, i)
                IfCondition EqualTo(ch, 10) ThenBlock: {
                    // Line starts at scan_pos + i + 1
                    line_start = Add(scan_pos, Add(i, 1))
                    // Store offset (2 bytes LE — file offsets fit in 32-bit for sane logs)
                    off_idx = Multiply(found, 4)
                    SetByte(offsets, off_idx,     BitwiseAnd(line_start, 255))
                    SetByte(offsets, Add(off_idx,1), BitwiseAnd(RightShift(line_start,8),255))
                    SetByte(offsets, Add(off_idx,2), BitwiseAnd(RightShift(line_start,16),255))
                    SetByte(offsets, Add(off_idx,3), BitwiseAnd(RightShift(line_start,24),255))
                    found = Add(found, 1)
                }
                i = Subtract(i, 1)
            }
        }

        // Now read and render lines in forward order (oldest first)
        // offsets[found-1] is oldest, offsets[0] is newest
        render_row = UILayout.chat_top
        j = Subtract(found, 1)
        WhileLoop And(GreaterEqual(j, 0), LessThan(render_row, UILayout.chat_bot)) {
            off_idx = Multiply(j, 4)
            lo0 = GetByte(offsets, off_idx)
            lo1 = GetByte(offsets, Add(off_idx, 1))
            lo2 = GetByte(offsets, Add(off_idx, 2))
            lo3 = GetByte(offsets, Add(off_idx, 3))
            line_off = Add(lo0, Add(Multiply(lo1,256), Add(Multiply(lo2,65536), Multiply(lo3,16777216))))

            SystemCall(8, rfd, line_off, 0)
            bytes = SystemCall(0, rfd, ChatLog_State.read_buf, ChatLog_Config.READ_BUF)

            // Extract "t" field value from NDJSON line and render it
            // Also extract "r" field for color selection
            ChatLog_RenderLine(render_row, ChatLog_State.read_buf, bytes)
            render_row = Add(render_row, 1)
            j = Subtract(j, 1)
        }

        SystemCall(3, rfd)   // close read fd
        Deallocate(offsets, Multiply(n_lines, 8))
        ReturnValue(found)
    }
}
```

### ChatLog_RenderLine

Parses one NDJSON line from a raw buffer, extracts `r` and `t`
fields, colors by role, prints clipped to `chat_w`.

```ailang
Function.ChatLog_RenderLine {
    Input: row: Integer
    Input: buf: Address
    Input: len: Integer
    Body: {
        // Quick parse: find "r":"X" and "t":"..."
        // Scan for :"r":" pattern, read next byte as role
        // Scan for :"t":" pattern, read until closing "
        // Do NOT use Library.JSON — parse inline to avoid allocation

        role = 97   // default 'a' assistant
        text_start = 0
        text_len   = 0

        i = 0
        WhileLoop LessThan(i, Subtract(len, 6)) {
            // Look for "r":"
            IfCondition And(
                EqualTo(GetByte(buf, i),     34),
                EqualTo(GetByte(buf, Add(i,1)), 114),
                EqualTo(GetByte(buf, Add(i,2)), 34),
                EqualTo(GetByte(buf, Add(i,3)), 58),
                EqualTo(GetByte(buf, Add(i,4)), 34)
            ) ThenBlock: {
                role = GetByte(buf, Add(i, 5))
            }
            // Look for "t":"
            IfCondition And(
                EqualTo(GetByte(buf, i),     34),
                EqualTo(GetByte(buf, Add(i,1)), 116),
                EqualTo(GetByte(buf, Add(i,2)), 34),
                EqualTo(GetByte(buf, Add(i,3)), 58),
                EqualTo(GetByte(buf, Add(i,4)), 34)
            ) ThenBlock: {
                text_start = Add(i, 5)
                // Find closing " (unescaped)
                k = text_start
                WhileLoop LessThan(k, len) {
                    ck = GetByte(buf, k)
                    IfCondition EqualTo(ck, 92) ThenBlock: {
                        k = Add(k, 2)  // skip escaped char
                    } ElseBlock: {
                        IfCondition EqualTo(ck, 34) ThenBlock: {
                            text_len = Subtract(k, text_start)
                            BreakLoop
                        }
                        k = Add(k, 1)
                    }
                }
            }
            i = Add(i, 1)
        }

        // Color by role
        IfCondition EqualTo(role, 117) ThenBlock: {   // 'u' user
            TUI_SetFG(TUI_Colors.WHITE)
            TUI_Bold()
        }
        IfCondition EqualTo(role, 97) ThenBlock: {    // 'a' assistant
            TUI_ResetAttr()
        }
        IfCondition EqualTo(role, 116) ThenBlock: {   // 't' tool
            TUI_SetFG(TUI_Colors.GREEN)
        }
        IfCondition EqualTo(role, 115) ThenBlock: {   // 's' system/separator
            TUI_SetFG(TUI_Colors.BLACK)
            TUI_Bold()
        }

        // Print clipped to chat pane width
        TUI_MoveTo(row, UILayout.chat_col)
        TUI_Draw_PrintClipped(row, UILayout.chat_col,
            Add(buf, text_start), UILayout.chat_w)
        TUI_ResetAttr()

        ReturnValue(0)
    }
}
```

---

## Step 2 — Rotation on session start and /clear

In `ClaudeCode.ailang`, at session start before `ChatLog_Init`:

```ailang
// Rotate: rename chat_current.ndjson → chat_TIMESTAMP.ndjson
// Then prune: keep only KEEP_FILES most recent chat_*.ndjson files
ChatLog_Rotate()
```

`ChatLog_Rotate` — implement in `Library.ChatLog.ailang`:
1. Build timestamp string from `clock_gettime`
2. Rename `chat_current.ndjson` to `chat_NNNN.ndjson` using
   syscall 82 (`rename(old, new)`)
3. List `~/.halcode/` with `getdents` (syscall 78), count
   `chat_*.ndjson` files, unlink oldest if count > KEEP_FILES

On `/clear` command: call `ChatLog_Rotate()` then `ChatLog_Init()`
to start a fresh log.

---

## Step 3 — Wire AppendLine into UI_ChatPrint

In `UI.ailang`, every place text is written to the chat area,
add a `ChatLog_Append` call before the terminal write.

Role codes:
- User message prefix `> ...` → role = 117 ('u')
- Assistant response text → role = 97 ('a')
- Tool summary `✓ N tools executed` → role = 116 ('t')
- Turn separator line → role = 115 ('s')

For multi-line responses, append one log entry per visual line
(i.e. after each `\n` in the stream, flush the accumulated line
to the log). Track a line accumulation buffer in `UI_State`:

```
"line_acc":     Initialize=0   // allocated 512 bytes
"line_acc_pos": Initialize=0
```

In `UI_ChatPrint`, accumulate chars into `line_acc`. On `\n`
or column overflow: call `ChatLog_Append(97, line_acc)`,
reset `line_acc_pos = 0`.

---

## Step 4 — Scroll back input handling

In `UI.ReadLine` (idle input only — NOT during streaming):

```ailang
IfCondition EqualTo(key, TUI_Keys.KEY_PGUP) ThenBlock: {
    // Move scroll anchor back by chat_rows lines
    // Implemented as: subtract chat_rows worth of bytes
    // by scanning backward that many newlines from current anchor
    UI_ScrollBack(Subtract(UILayout.chat_rows, 2))
    TUI_Flush()
}
IfCondition EqualTo(key, TUI_Keys.KEY_PGDN) ThenBlock: {
    UI_ScrollForward(Subtract(UILayout.chat_rows, 2))
    TUI_Flush()
}
```

`UI_ScrollBack(n)`:
- Scan backward from `ChatLog_State.file_pos` counting `n` newlines
- Update `ChatLog_State.file_pos` to that position
- Call `ChatLog_ReadBack(chat_rows)` to repaint

`UI_ScrollForward(n)`:
- Scan forward from `ChatLog_State.file_pos` counting `n` newlines
- Cap at actual file end (do not go past EOF)
- Update `ChatLog_State.file_pos`
- Call `ChatLog_ReadBack(chat_rows)` to repaint

On new user turn: reset `ChatLog_State.file_pos` to EOF
(auto-scroll to bottom).

Add a visual scroll indicator — when `file_pos < EOF`, show
`[SCROLL]` in dim red at the right of the hint row so the user
knows they are not at the live view.

---

## Step 5 — Remove old scroll region approach

Once `ChatLog_ReadBack` is the render source:

1. Remove `TUI_SetScrollRegion` and `TUI_ClearScrollRegion` calls
   from the chat print path. The terminal no longer drives scrolling.
2. Chat area is now fully repainted by `ChatLog_ReadBack` on each
   new line. This also fixes the divider vanishing bug — the divider
   is drawn by `UI_DrawChrome` which is called after each repaint.
3. Keep `TUI_Layout_Activate` / `TUI_Layout_Deactivate` only where
   they guard non-chat draws (hint row, rule row, chrome).

---

## Build and verify

```bash
cd /path/to/AILangSH && bash build.sh
```

1. Launch. Send "hello". Check `~/.halcode/chat_current.ndjson`
   exists and contains valid NDJSON lines:
   ```bash
   cat ~/.halcode/chat_current.ndjson
   ```
2. Each line is valid JSON with `r`, `t`, `ts` fields.
3. Send a long multi-turn conversation. Page Up scrolls back
   through earlier turns. Page Down returns to latest.
4. `[SCROLL]` indicator appears when not at bottom, disappears
   when scrolled back to latest.
5. `/clear` rotates the log file. Old file renamed with timestamp.
6. Restart HalCode — previous session's log preserved as
   `chat_NNNN.ndjson` in `~/.halcode/`.
7. Divider stays visible through all scroll events.

## Hard constraints

- `ChatLog_State.fd` append fd is NEVER used for reading.
  Always open a separate read-only fd for scroll operations.
- `ChatLog_Rotate` must not fail silently if rename fails
  (file may not exist on first run — check fd before rename).
- Scroll is disabled during streaming. `file_pos` is locked
  to EOF while `UI_State.streaming = 1`.
- `Library.ChatLog.ailang` must stay under 300 LOC.
  If it grows past 300, split rotation logic into
  `Library.ChatLog.Rotate.ailang`.
- Do not modify `History.ailang`, `Anthropic.ailang`, `cc_tools/*`.
