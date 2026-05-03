# Session: Grep BUG-3 through BUG-7 Fixes

**Date:** 2026-05-16
**File:** `cc_tools/cc_grep_ipc.ailang`
**Session:** Fixes 3-7 from `docs/grep-bug-hitlist.md` (BUG-8, BUG-9 were re-inspected as non-bugs)

---

## BUG-2 :: Match_RegexDFA return value misinterpretation

**Lines:** 431–440
**Fix:** `EqualTo(d, 1)` → `GreaterEqual(d, 0)`. Removed dead NFA fallback and `EqualTo(d, 0)` false-negative.
`Regex_DFASearch` returns match position ≥0 on success, -1 on no match. Old code treated pos=0 as "no match" and only caught pos=1.

```ailang
// OLD:
IfCondition EqualTo(d, 1) ThenBlock: { ReturnValue(1) }
IfCondition EqualTo(d, 0) ThenBlock: { ReturnValue(0) }
rhandle = Dereference(Patterns.regex)
ReturnValue(Regex_Search(rhandle, line, line_len))

// NEW:
IfCondition GreaterEqual(d, 0) ThenBlock: { ReturnValue(1) }
ReturnValue(0)
```

---

## BUG-3 :: CCGrep_ProcessDir skips DT_UNKNOWN entries

**Lines:** 928–948 (dtype dispatch)
**Fix:** Added `Or(EqualTo(dtype, CCGrepSys.DT_REG), EqualTo(dtype, 0))` so DT_UNKNOWN (filesystems without d_type support: NFS, FUSE, older XFS) are attempted as regular files.

```ailang
// OLD: IfCondition EqualTo(dtype, CCGrepSys.DT_REG) ThenBlock: {
// NEW:
IfCondition Or(EqualTo(dtype, CCGrepSys.DT_REG), EqualTo(dtype, 0)) ThenBlock: {
```

---

## BUG-4 :: SearchBoyerMoore never uses bad-char skip table

**Lines:** 344 (`pos = Add(cand, 1)`)
**Fix:** On MemCompare mismatch, consult bad-char table using the character at `cand + plen - 1` (last byte of attempted match). Guard skip=0 → skip=1 to prevent infinite loops.

```ailang
// OLD:
pos = Add(cand, 1)

// NEW:
skip = GetByte(table, GetByte(line, Add(cand, Subtract(plen, 1))))
IfCondition EqualTo(skip, 0) ThenBlock: { skip = 1 }
pos = Add(cand, skip)
```

---

## BUG-5 :: Memory leak — pattern buffers never freed between requests

**Lines:** New function `CCGrep_FreePatterns` added before `CCGrep_InitPatterns` (~line 590). Called at request start (~line 1107) instead of bare `Patterns.count = 0`.

Frees per-pattern: `ptrs` (own string), `lc_ptrs` (lowercase copy), `regex` handle, `bc_tables`, `rx_prefix`, `rx_prefix_bc`. Resets `Patterns.count = 0`.

```ailang
Function.CCGrep_FreePatterns {
    Body: {
        i = 0
        WhileLoop LessThan(i, Patterns.count) {
            // free each per-pattern allocation
            i = Add(i, 1)
        }
        Patterns.count = 0
    }
}
```

---

## BUG-6 :: CCGrep_AppendUInt allocation churn

**Lines:** 163–182
**Fix:** Uses pre-allocated `Scratch.digits` (24 bytes, allocated once at startup) instead of `Allocate(24)`/`Deallocate(digits, 24)` on every call.

- Added `"digits": Initialize=0, CanChange=True` to `FixedPool.Scratch`
- Added `Scratch.digits = Allocate(24)` in main init
- Replaced `digits` local variable with `Scratch.digits`

---

## BUG-7 :: include filter silently ignored when recursive=0

**Lines:** 1174 (non-recursive file path)
**Fix:** Added basename extraction + `CCGrep_GlobMatch` check before opening file. If include filter set and basename doesn't match, returns `"(no matches — excluded by include filter)"`.

```ailang
IfCondition NotEqual(GrepConfig.include_pat, 0) ThenBlock: {
    // Extract basename from path (scan for last '/')
    // Check CCGrep_GlobMatch(bname, GrepConfig.include_pat)
    // Skip if no match
}
```

---

## Build

```bash
cd /mnt/c/Users/Sean/Documents/AiLangSH
./ailang.x Applications/HalCode9000/cc_tools/cc_grep_ipc.ailang /tmp/hal_cc_grep_ipc_fixes.x
```

- **Analyzer:** 0 errors
- **Build:** `[ok]` — 154,811 bytes (was 154,767, +44 bytes)
- **Binary:** `/tmp/hal_cc_grep_ipc_fixes.x`

---

## Deployment

The running `cc_grep_ipc.x` daemon (PID 590005) locks the file. To deploy:

```bash
# 1. Stop the halcode engine (kills all cc_*_ipc daemons)
# 2. cp /tmp/hal_cc_grep_ipc_fixes.x Applications/HalCode9000/cc_grep_ipc.x
# 3. Restart
```

---

## Status
- [x] BUG-2: Match_RegexDFA DFA pos-0 false-negative — FIXED
- [x] BUG-3: DT_UNKNOWN entries skipped — FIXED
- [x] BUG-4: Boyer-Moore skip table unused — FIXED
- [x] BUG-5: Per-request memory leak — FIXED (CCGrep_FreePatterns)
- [x] BUG-6: AppendUInt allocation churn — FIXED (Scratch.digits)
- [x] BUG-7: include filter ignored w/o recursive — FIXED
- [x] BUG-8: Re-inspected — no bug (newline handling correct)
- [x] BUG-9: Re-inspected — no bug (d_reclen parsing correct)
- [x] Analyzer: 0 errors
- [x] Build: passes
- [ ] Binary deployed (requires daemon restart)
- [ ] Runtime tested
