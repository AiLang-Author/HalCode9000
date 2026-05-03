# Session: Grep BUG-1 Fix — Directory/File Dispatch

**Bug:** Recursive mode fails silently on file paths; non-recursive on directory returns "(no matches)" instead of error.

**File to edit:** `cc_tools/cc_grep_ipc.ailang`
**Function:** `CCGrep_HandleRequest` — the dispatch block
**Lines:** ~1070–1090 (the `IfCondition EqualTo(GrepConfig.recursive, 1)` block)

## Current Code (problematic)

```ailang
IfCondition EqualTo(GrepConfig.recursive, 1) ThenBlock: {
    CCGrep_ProcessDir(path, 0)
} ElseBlock: {
    fd = SystemCall(CCGrepSys.SYS_OPENAT, CCGrepSys.AT_FDCWD, path, 0)
    IfCondition LessThan(fd, 0) ThenBlock: {
        CCGrep_SendError(client_fd, id, StringConcat("cannot open: ", path))
        JSON.Free(parsed)  ReturnValue(0)
    }
    GrepConfig.show_prefix = 0
    CCGrep_ProcessStream(fd, path)
    SystemCall(CCGrepSys.SYS_CLOSE, fd, 0, 0, 0, 0)
}
```

## Fix Plan

Replace with a unified path-type detection that tries O_DIRECTORY first, then falls back to O_RDONLY:

1. Try `openat(O_DIRECTORY)` — if success: it's a directory
   - If `recursive=1`: call `CCGrep_ProcessDir`
   - If `recursive=0`: close fd, send error "is a directory; set recursive=1"
2. If `openat(O_DIRECTORY)` fails: try `openat(O_RDONLY)` — it's a file (or missing)
   - Process as single file regardless of `recursive` flag

## Build Command
```bash
cd /mnt/c/Users/Sean/Documents/AiLangSH
bash build.sh
```
Or targeted:
```bash
cd /mnt/c/Users/Sean/Documents/AiLangSH
./ailang.x Applications/HalCode9000/cc_tools/cc_grep_ipc.ailang Applications/HalCode9000/cc_grep_ipc.x
```

## Verification Strategy
After build, test with JSON over the socket:
```python
# Test 1: file path (should find matches)
# Test 2: directory path with recursive=0 (should error)
# Test 3: directory path with recursive=1 (should recurse)
# Test 4: file path with recursive=1 (should still work — fallback to single-file)
```

## Status
- [x] Fix applied
- [x] Build passes (`[ok] hal_cc_grep_ipc`)
- [ ] Runtime tested (requires restart to pick up new binary)

## Build Result
```
build.sh --tools-only  →  [ok] hal_cc_grep_ipc
Install skipped (binaries running). Re-run after hal restart.
```
