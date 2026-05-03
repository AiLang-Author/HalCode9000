# Session: Grep BUG-2 Fix — After-Context Overflow Silent

**Bug:** After-context lines overflow `result_buf` without setting `stop_early`, causing silent truncation.

**File:** `cc_tools/cc_grep_ipc.ailang`
**Function:** `CCGrep_ProcessStream` — the non-matching-line path (ElseBlock)
**Lines:** ~823 (after-context emit block)

## Root Cause

In the stream processor's non-matching branch, after-context lines are emitted without checking `CCGrepState.truncated`. Compare:

**Match path (correct):**
```ailang
CCGrep_EmitLine(line_buf, line_len, line_no, label, 58)
CCGrepCtx.after_rem = GrepConfig.after_ctx
...
IfCondition GreaterThan(CCGrepState.truncated, 0) ThenBlock: {
    stop_early = 1
}
```

**After-context path (BUGGY — no truncated check):**
```ailang
IfCondition GreaterThan(CCGrepCtx.after_rem, 0) ThenBlock: {
    CCGrep_EmitLine(line_buf, line_len, line_no, label, 45)  // '-'
    CCGrepCtx.after_rem = Subtract(CCGrepCtx.after_rem, 1)
    // >>> MISSING: truncated check <<<
}
```

**Consequence:** When `RESULT_CAP` (512KB) fills up during after-context emission, `CCGrep_Append`/`CCGrep_AppendByte` set `CCGrepState.truncated = 1` but the stream processor keeps emitting lines. All subsequent output is silently discarded. The response includes `"truncated": true` but the user doesn't know which results were lost.

## Fix

Added `stop_early = 1` when `truncated` is true after emitting an after-context line:

```ailang
IfCondition GreaterThan(CCGrepCtx.after_rem, 0) ThenBlock: {
    CCGrep_EmitLine(line_buf, line_len, line_no, label, 45)  // '-'
    CCGrepCtx.after_rem = Subtract(CCGrepCtx.after_rem, 1)
    IfCondition GreaterThan(CCGrepState.truncated, 0) ThenBlock: {
        stop_early = 1
    }
}
```

This mirrors the pattern used after match-line emission.

## Build

```bash
build.sh --tools-only --no-copy  →  [ok] hal_cc_grep_ipc
```

New binary at `/tmp/hal_cc_grep_ipc.x` (154767 bytes). Replace `cc_grep_ipc.x` after hal restart.

## Deployment

The old binary is locked ("Text file busy") because the grep daemon is running. To deploy:

1. Stop hal / cc_grep_ipc daemon
2. `cp /tmp/hal_cc_grep_ipc.x Applications/HalCode9000/cc_grep_ipc.x`
3. Restart

## Verification

After deployment, test with a file that produces enough output to fill the 512KB result buffer. Before the fix, after-context lines would overflow silently. After the fix, `stop_early=1` halts processing and the response correctly includes `"truncated": true`.

## Status
- [x] Fix applied (1 line added)
- [x] Analyzer: 0 errors
- [x] Build passes: `[ok] hal_cc_grep_ipc`
- [ ] Binary deployed (locked by running daemon)
- [ ] Runtime tested
