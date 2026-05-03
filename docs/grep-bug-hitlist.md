# cc_grep_ipc.ailang — Bug Hit List

> Analysis of `cc_tools/cc_grep_ipc.ailang` (1222 LOC) vs. observed failures during documentation-editing session.
> Code is a socket-based IPC tool daemon forked from `AiLang_CoreUtils/dist/grep_util/grep.ailang`.

---

## CRITICAL — Correctness Bugs

### BUG-1 :: Recursive mode fails silently on regular-file paths

**Location:** `CCGrep_HandleRequest` dispatch logic (≈line 1070-1090)

**Code:**
```ailang
IfCondition EqualTo(GrepConfig.recursive, 1) ThenBlock: {
    CCGrep_ProcessDir(path, 0)     // <-- ALWAYS tries O_DIRECTORY
} ElseBlock: {
    fd = SystemCall(CCGrepSys.SYS_OPENAT, CCGrepSys.AT_FDCWD, path, 0)
    ...
}
```

**Root Cause:** `CCGrep_ProcessDir` opens with `O_RDONLY|O_DIRECTORY` (flags=65536). When `path` is a regular file, `openat()` returns an error because you cannot open a regular file with `O_DIRECTORY`. The function returns 0 silently, and `CCGrep_HandleRequest` falls through to send `"(no matches)"`.

**Observed Failure:** Calling `Grep(pattern="WinColor", path=".../Librarys/Display/Window")` with the implicit default `recursive=0` — if `Window` is a directory, `openat(O_RDONLY)` succeeds on Linux (directories ARE readable), then `SYS_READ` returns EISDIR, and the stream processor exits with 0 matches. Silent "(no matches)" instead of an error or automatic recursion.

**User-facing symptom:** `Grep` tool returns `(no matches)` when it should either:
- Return matches from the file, or
- Return an error: "path is a directory, set recursive=1"

**Fix:**
```ailang
// In CCGrep_HandleRequest, before dispatching:
IfCondition EqualTo(GrepConfig.recursive, 1) ThenBlock: {
    // Try as directory first
    oflags = BitwiseOr(CCGrepSys.O_RDONLY, CCGrepSys.O_DIRECTORY)
    test_fd = SystemCall(CCGrepSys.SYS_OPENAT, CCGrepSys.AT_FDCWD, path, oflags, 0, 0)
    IfCondition GreaterEqual(test_fd, 0) ThenBlock: {
        SystemCall(CCGrepSys.SYS_CLOSE, test_fd, 0, 0, 0, 0)
        CCGrep_ProcessDir(path, 0)
    } ElseBlock: {
        // Path is a regular file (or doesn't exist) — process as single file
        fd = SystemCall(CCGrepSys.SYS_OPENAT, CCGrepSys.AT_FDCWD, path, 0)
        IfCondition LessThan(fd, 0) ThenBlock: {
            CCGrep_SendError(client_fd, id, StringConcat("cannot open: ", path))
            JSON.Free(parsed)  ReturnValue(0)
        }
        GrepConfig.show_prefix = 0
        CCGrep_ProcessStream(fd, path)
        SystemCall(CCGrepSys.SYS_CLOSE, fd, 0, 0, 0, 0)
    }
}
```

---

### BUG-2 :: `Match_RegexDFA` misinterprets `Regex_DFASearch` return value

**Location:** `Match_RegexDFA` function (≈line 335)

**Code:**
```ailang
Function.Match_RegexDFA {
    ...
    Body: {
        d = Regex_DFASearch(line, line_len)
        IfCondition EqualTo(d, 1) ThenBlock: { ReturnValue(1) }   // only catches position==1
        IfCondition EqualTo(d, 0) ThenBlock: { ReturnValue(0) }   // treats position==0 as "no match"!
        rhandle = Dereference(Patterns.regex)
        ReturnValue(Regex_Search(rhandle, line, line_len))        // fallback NFA for all else
    }
}
```

**Root Cause:** `Regex_DFASearch` returns a **match position** (>= 0) on success, or -1 on no match. Evidence: `RegexLineSearch` checks `GreaterEqual(d, 0)`. But `Match_RegexDFA`:
1. Checks for `d == 1` — only catches matches at byte position 1
2. Checks for `d == 0` — treats match at position 0 as "no match" (FALSE NEGATIVE)
3. Falls through to NFA for all other positions (correctness preserved but DFA optimization wasted)

**Severity:** A pattern matching at byte-position 0 of a line will be **silently missed** when the DFA strategy is active.

**Fix:**
```ailang
d = Regex_DFASearch(line, line_len)
IfCondition GreaterEqual(d, 0) ThenBlock: { ReturnValue(1) }  // match at any position
ReturnValue(0)                                                  // no match
```

---

## HIGH — Directory/File Walk Bugs

### BUG-3 :: `CCGrep_ProcessDir` skips DT_UNKNOWN and DT_LNK entries

**Location:** `CCGrep_ProcessDir` entry-type dispatch (≈line 930)

**Code:**
```ailang
IfCondition EqualTo(dtype, CCGrepSys.DT_DIR) ThenBlock: {
    CCGrep_ProcessDir(child_buf, Add(depth, 1))
} ElseBlock: {
    IfCondition EqualTo(dtype, CCGrepSys.DT_REG) ThenBlock: {
        ...  // only DT_REG (8) processed
    }
    // DT_UNKNOWN (0), DT_LNK (10), DT_FIFO, DT_SOCK, DT_CHR, DT_BLK — all silently skipped
}
```

**Root Cause:** The `ElseBlock` only processes `DT_REG` (8). On filesystems that don't support `d_type` (e.g., some NFS mounts, older XFS without `ftype=1`, FUSE filesystems), **all** entries return `d_type=DT_UNKNOWN` (0). The entire directory tree is silently empty. Symlinks to regular files (`DT_LNK`) are also skipped.

**Fix:** Add a fallback for `DT_UNKNOWN` that attempts `openat(O_RDONLY)` and processes the file if the open succeeds (the kernel will fail if it's not a regular file). Optionally handle `DT_LNK` by following the symlink.

```ailang
} ElseBlock: {
    // Try DT_REG or DT_UNKNOWN (filesystem doesn't support d_type)
    IfCondition Or(EqualTo(dtype, CCGrepSys.DT_REG), EqualTo(dtype, 0)) ThenBlock: {
        ...
    }
}
```

---

## MEDIUM — Performance / Resource Bugs

### BUG-4 :: `SearchBoyerMoore` never uses the bad-character skip table

**Location:** `SearchBoyerMoore` function (≈line 260)

**Code:** The function accepts a `table` parameter (the bad-character skip table) but never references it. The algorithm:
1. Uses `MemChr` to find the first occurrence of `pat[0]`
2. Verifies full match with `MemCompare`
3. On mismatch: **advances by 1** (`pos = Add(cand, 1)`) instead of skipping by `table[text[mismatch_pos]]`

**Impact:** Not Boyer-Moore — it's a naive O(n·m) search with a first-byte filter. The bad-char table is built in `CCGrep_CompilePatterns` (wasting CPU), passed to `SearchBoyerMoore`, and ignored.

**Fix:** After a `MemCompare` failure, consult the bad-char table:
```ailang
// On mismatch at position 'j' in pattern:
bad_char_skip = GetByte(table, GetByte(line, Add(cand, j)))
pos = Add(cand, bad_char_skip)
```

Or minimally, rename the function to `SearchFirstByteFilter` to avoid confusion.

---

### BUG-5 :: Memory leak — pattern buffers never freed between requests

**Location:** `CCGrep_AddPattern` allocations + `CCGrep_CompilePatterns` (≈line 600-650)

**Code:**
```ailang
Function.CCGrep_AddPattern {
    ...
    own = Allocate(Add(plen, 1))
    lc  = Allocate(Add(plen, 1))
    ...
}
```

**Impact:** Each grep request allocates pattern strings, lowercased copies, regex handles, prefix buffers, and bad-char tables. `Patterns.count` is reset to 0 between requests, but old pointers are simply overwritten — no `Deallocate` calls. The `cc_grep_ipc.x` daemon leaks memory linearly with each request.

**Severity:** Medium for long sessions (gradual OOM on the daemon). Not user-visible until the daemon dies.

**Fix:** Add a `CCGrep_FreePatterns` function called at the top of each request:
```ailang
Function.CCGrep_FreePatterns {
    Body: {
        i = 0
        WhileLoop LessThan(i, Patterns.count) {
            Deallocate(Dereference(Add(Patterns.ptrs, Multiply(i, 8))), 0)
            Deallocate(Dereference(Add(Patterns.lc_ptrs, Multiply(i, 8))), 0)
            // ... free regex handles, prefix buffers, bc tables
            i = Add(i, 1)
        }
    }
}
```

---

### BUG-6 :: `CCGrep_AppendUInt` hardcodes digit buffer size at 24 bytes

**Location:** `CCGrep_AppendUInt` (≈line 140)

**Code:**
```ailang
digits = Allocate(24)
```

**Impact:** 64-bit unsigned max is 18446744073709551615 (20 digits). 24 bytes is enough. But `Deallocate(digits, 24)` on every number output is allocation-heavy. This is a minor perf issue, not a correctness bug.

**Fix:** Use a stack-allocated scratch buffer or a FixedPool pre-allocation.

---

## LOW — Edge Cases / Polish

### BUG-7 :: `include` filter silently ignored when `recursive=0`

**Location:** `CCGrep_HandleRequest` — the non-recursive path never checks `GrepConfig.include_pat`

**Impact:** If a user passes `include="*.ailang"` with `recursive=0` and a non-.ailang file, the file is searched anyway. The include filter is only applied in `CCGrep_ProcessDir`. This is confusing but not dangerous.

**Fix:** Either warn when include is set without recursive, or apply the glob check before the non-recursive file open.

---

### BUG-8 :: No newline at end of tool call response line

**Location:** `CCGrep_EmitLine` (≈line 240)

**Code:**
```ailang
CCGrep_Append(line, line_len)
CCGrep_AppendByte(10)
```

**Impact:** Lines end with `\n` (10). This is correct. But the final result buffer is NUL-terminated for JSON serialization. No bug here upon re-inspection — the `CCGrep_SendResult` sends `CCGrepState.result_buf` which is already properly formatted.

---

### BUG-9 :: `d_reclen` parsing uses `Add()` for 16-bit LE recombination

**Location:** `CCGrep_ProcessDir` getdents64 parsing (≈line 905)

**Code:**
```ailang
reclen = Add(GetByte(entry, 16), Multiply(GetByte(entry, 17), 256))
```

**Impact:** This is `low_byte + high_byte * 256`, which correctly reconstructs a little-endian uint16. No bug.

---

## Summary

| Bug | Severity | Type | Observable? |
|-----|----------|------|-------------|
| BUG-1 | CRITICAL | Recursive=1 on file path → silent no-match | YES — Failure 1 & 3 |
| BUG-2 | CRITICAL | DFA match-at-pos-0 treated as no-match | Rarely (regex must match at line start) |
| BUG-3 | HIGH | DT_UNKNOWN/DT_LNK entries skipped | On NFS/FUSE filesystems |
| BUG-4 | MEDIUM | Boyer-Moore table built but unused | Performance only |
| BUG-5 | MEDIUM | Per-request memory leak | Long-running daemon OOM |
| BUG-6 | LOW | Allocation churn in AppendUInt | Performance only |
| BUG-7 | LOW | include filter ignored w/o recursive | Confusing but harmless |
| BUG-8 | NONE | Re-inspected, correct | — |
| BUG-9 | NONE | Re-inspected, correct | — |

**Fixes for the observed failures (Failure 1, 2, 3):**

- **Failure 1** (`Grep` on directory path): BUG-1. Fix the dispatch to detect directories and either auto-enable recursion or return an error.
- **Failure 2** (`grep -r` via Bash → "unknown option"): The system `grep` binary is intercepted by the AILang grep, which doesn't support `-r`. This is an environment issue, not this code.
- **Failure 3** (`recursive=1` + `include` filter on directory): BUG-1 or BUG-3. If the path resolved to a regular file, BUG-1 applies. If the filesystem doesn't support d_type, BUG-3 applies.
