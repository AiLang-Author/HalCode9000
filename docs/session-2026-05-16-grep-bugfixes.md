# Session: Grep IPC — Bug Fixes BUG-1 through BUG-9

**Date:** 2026-05-16
**File:** `cc_tools/cc_grep_ipc.ailang`
**Git base:** `390974b` (UI prompt stability fixes; grep BUG-1 patch)
**Working tree:** uncommitted changes for BUG-2 through BUG-7 (see `git diff`)

---

## Overview

Audited the grep IPC worker (`cc_grep_ipc.ailang`) against `docs/grep-bug-hitlist.md`.
Found and fixed 7 real bugs (BUG-1 through BUG-7). BUG-8 and BUG-9 were re-inspected
and found to be correct — no fix needed.

---

## Fixes Applied

### BUG-1 :: DFA not used for patterns with leading wildcard prefix
**Commit:** `390974b` (already committed)
**Issue:** Regex_DFASearch was never called for patterns like `.*foo` or `foo.*bar`
because the DFA-eligibility check was too strict.
**Fix:** Relaxed the check to allow DFA for patterns with leading/trailing `.*`.

---

### BUG-2 :: Match_RegexDFA return value misinterpretation
**Lines:** 431–440
**Issue:** `Regex_DFASearch` returns match position ≥0 on success, -1 on failure.
Old code treated pos=0 as "no match" (`EqualTo(d, 0)` → return 0) and only caught
pos=1 (`EqualTo(d, 1)` → return 1). Also had dead NFA fallback code.
**Fix:** Changed to `GreaterEqual(d, 0)` → return 1, else return 0. Removed dead code.
```ailang
// OLD:
IfCondition EqualTo(d, 1) ThenBlock: { ReturnValue(1) }
IfCondition EqualTo(d, 0) ThenBlock: { ReturnValue(0) }
// ...dead NFA fallback...

// NEW:
IfCondition GreaterEqual(d, 0) ThenBlock: { ReturnValue(1) }
ReturnValue(0)
```

---

### BUG-3 :: CCGrep_ProcessDir skips DT_UNKNOWN entries
**Lines:** 928–948
**Issue:** Filesystems without `d_type` support (NFS, FUSE, older XFS) return
`DT_UNKNOWN` (0). These entries were silently skipped.
**Fix:** Added `Or(EqualTo(dtype, DT_REG), EqualTo(dtype, 0))` so DT_UNKNOWN
entries are attempted as regular files.
```ailang
IfCondition Or(EqualTo(dtype, CCGrepSys.DT_REG), EqualTo(dtype, 0)) ThenBlock: {
```

---

### BUG-4 :: SearchBoyerMoore never uses bad-char skip table
**Lines:** 344
**Issue:** On mismatch, `pos = Add(cand, 1)` always advances by 1, never consulting
the Boyer-Moore bad-character table.
**Fix:** Consult bad-char table using the last byte of the attempted match position.
Guard skip=0 → skip=1 to prevent infinite loops.
```ailang
skip = GetByte(table, GetByte(line, Add(cand, Subtract(plen, 1))))
IfCondition EqualTo(skip, 0) ThenBlock: { skip = 1 }
pos = Add(cand, skip)
```

---

### BUG-5 :: Memory leak — pattern buffers never freed between requests
**Lines:** New function `CCGrep_FreePatterns` (~line 590). Called at request start
(~line 1107) instead of bare `Patterns.count = 0`.
**Issue:** Per-pattern allocations (ptrs, lc_ptrs, regex handles, bc_tables,
rx_prefix, rx_prefix_bc) accumulated across requests with no free path.
**Fix:** `CCGrep_FreePatterns()` walks all patterns, frees each owned allocation
(Deallocate/Regex_Free), then resets count.
```ailang
Function.CCGrep_FreePatterns {
    // Free: ptrs, lc_ptrs, regex, bc_tables, rx_prefix, rx_prefix_bc per pattern
    Patterns.count = 0
}
```

---

### BUG-6 :: CCGrep_AppendUInt allocation churn
**Lines:** 163–182
**Issue:** `Allocate(24)`/`Deallocate(digits, 24)` on every call for digit buffer.
**Fix:** Uses pre-allocated `Scratch.digits` (24 bytes, allocated once at startup).
- Added `"digits": Initialize=0, CanChange=True` to `FixedPool.Scratch`
- Added `Scratch.digits = Allocate(24)` in `SubRoutine.Main`
- Replaced `digits` local with `Scratch.digits`

---

### BUG-7 :: `include` filter silently ignored when `recursive=0`
**Lines:** ~1174 (non-recursive file path)
**Issue:** When `recursive=0` with `include` filter set, the file was opened and
scanned regardless of the filter.
**Fix:** Extract basename from path (scan for last '/'), run `CCGrep_GlobMatch`.
If no match, return `"(no matches — excluded by include filter)"`.

---

### BUG-8 :: Re-inspected — NO BUG
**Issue examined:** Newline handling at end of file.
**Finding:** Correct. `CCGrep_ReadLine` properly handles EOF-at-newline case.

---

### BUG-9 :: Re-inspected — NO BUG
**Issue examined:** `d_reclen` parsing in `CCGrep_ProcessDir`.
**Finding:** Correct. `struct linux_dirent64` parsing follows kernel convention.

---

## Build

```bash
cd /mnt/c/Users/Sean/Documents/AiLangSH
./ailang.x Applications/HalCode9000/cc_tools/cc_grep_ipc.ailang /tmp/hal_cc_grep_ipc_fixes.x
```

- **Analyzer:** 0 errors
- **Build:** passes — 154,811 bytes (+44 from previous 154,767)

---

## Git Status

```
 M cc_tools/cc_grep_ipc.ailang    ← BUG-2 through BUG-7 fixes (uncommitted)
?? docs/                           ← New session docs directory (untracked)
```

**Base commit:** `390974b UI prompt stability fixes; grep BUG-1 patch`
**On branch:** (check `git branch`)

---

## Deployment

```bash
# 1. Stop the halcode engine (kills all cc_*_ipc daemons)
# 2. cp /tmp/hal_cc_grep_ipc_fixes.x Applications/HalCode9000/cc_grep_ipc.x
# 3. Restart
```

---

## Status

| Bug | Description | Status |
|-----|-------------|--------|
| BUG-1 | DFA eligibility too strict | ✅ Committed (390974b) |
| BUG-2 | Match_RegexDFA pos-0 false-negative | ✅ Fixed (uncommitted) |
| BUG-3 | DT_UNKNOWN entries skipped | ✅ Fixed (uncommitted) |
| BUG-4 | Boyer-Moore skip table unused | ✅ Fixed (uncommitted) |
| BUG-5 | Per-request memory leak | ✅ Fixed (uncommitted) |
| BUG-6 | AppendUInt allocation churn | ✅ Fixed (uncommitted) |
| BUG-7 | Include filter ignored w/o recursive | ✅ Fixed (uncommitted) |
| BUG-8 | Newline handling | ✅ No bug |
| BUG-9 | d_reclen parsing | ✅ No bug |
| Build | Analyzer + compiler | ✅ Passes |
| Deploy | Binary in production | ⬜ Pending daemon restart |
| Runtime | Functional testing | ⬜ Not yet tested |
| Commit | Git commit | ⬜ All fixes uncommitted |
