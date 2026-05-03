# cc_grep_ipc.ailang — Feature Completeness Audit

> Measured against GNU grep 3.x. Category scores: ❌ missing, ⚠️ partial/buggy, ✅ complete.

---

## Matching Engines

| Feature | Status | Notes |
|---------|--------|-------|
| Literal (fixed-string) search | ✅ | Boyer-Moore BC table (built but unused — see BUG-4) |
| Case-insensitive literal | ✅ | `CompareAt` with `ToLowerByte` per-character |
| Regex (Thompson NFA) | ✅ | `LibraryImport.Regex_Thompson`, `Regex_Search` |
| Regex with prefix optimization | ✅ | BM on extracted literal prefix, then NFA verify |
| Regex insensitive | ✅ | Lowercases line into scratch buffer, then NFA |
| DFA optimization | ⚠️ | `Regex_DFAInit` + `Regex_DFASearch` — but BUG-2 breaks pos-0 matches |
| Whole-line match (`-x`) | ✅ | `Match_WholeLine` |
| Word match (`-w`) | ✅ | `Match_WordMatch` with `IsWordChar` boundary check |
| Invert match (`-v`) | ✅ | `GrepConfig.invert` flips result in stream processor |
| Multiple patterns | ✅ | Up to `MAX_PATTERNS` (256), OR semantics |
| Empty pattern | ✅ | STRATEGY.EMPTY → always matches |

---

## Output Control

| Feature | Status | Notes |
|---------|--------|-------|
| Line numbers (`-n`) | ✅ | `GrepConfig.line_numbers` |
| Filename prefix (`-H`) | ✅ | `GrepConfig.show_prefix` + label parameter |
| Suppress filename (`-h`) | ⚠️ | No explicit flag; set `show_prefix=0` (only done for non-recursive single-file) |
| Count only (`-c`) | ❌ | Not implemented. `CCGrepState.match_count` is tracked but never output standalone |
| Limit matches (`-m N`) | ✅ | `max_results`/`CCGrepState.max_results` |
| Context before (`-B N`) | ✅ | `GrepConfig.before_ctx` + `CCGrep_PushRing`/`CCGrep_EmitRing` circular buffer |
| Context after (`-A N`) | ✅ | `GrepConfig.after_ctx` + `CCGrepCtx.after_rem` counter |
| Context around (`-C N`) | ✅ | Sets both `before_ctx` and `after_ctx` |
| Group separator (`--`) | ✅ | Emitted when gap since last context group |
| Color/highlight | ❌ | No ANSI color on match text |
| Only matching part (`-o`) | ❌ | Not implemented |
| Quiet mode (`-q`) | ❌ | Not implemented |
| Line-buffered output | ❌ | Output is fully buffered in `result_buf`, sent once at end |

---

## File Selection

| Feature | Status | Notes |
|---------|--------|-------|
| Single file | ✅ | `openat(O_RDONLY)` → `ProcessStream` |
| Recursive directory (`-r`) | ⚠️ | `CCGrep_ProcessDir` with `getdents64` — but BUG-1, BUG-3 |
| Include glob (`--include`) | ⚠️ | `CCGrep_GlobMatch` for `*.suffix` and exact match. Works correctly in theory, but BUG-1/BUG-3 block it |
| Exclude glob (`--exclude`) | ❌ | Not implemented |
| Exclude-dir (`--exclude-dir`) | ❌ | Not implemented |
| Max depth (`--max-depth`) | ✅ | `CCGrepConst.MAX_DEPTH` (20) |
| Binary file detection | ✅ | `MemChr(read_buf, 0, n)` — NUL byte check on first chunk |
| Binary file handling (`-I`/`--binary-files`) | ⚠️ | Prints "Binary file X matches" and stops. No `without-match` or `text` modes |
| Symlink following | ❌ | DT_LNK entries skipped (BUG-3). No `-R`/`--dereference-recursive` |
| Device/FIFO skip | ⚠️ | Implicitly skipped via `dtype != DT_REG` check |
| Reading stdin (`-`) | ❌ | No stdin support; `path` is required |

---

## Regex Features

| Feature | Status | Notes |
|---------|--------|-------|
| Basic regex (`^$.*[]`) | ✅ | Via Thompson NFA |
| Extended regex (`+?{}()\|`) | ✅ | Via Thompson NFA |
| Perl-compatible regex | ❌ | Thompson NFA only; no backreferences, lookahead, etc. |
| Word boundaries (`\<`, `\>`) | ❌ | Use `-w` flag instead |
| Case-insensitive regex (`-i`) | ✅ | Via `Match_RegexInsensitive` |

---

## Performance

| Feature | Status | Notes |
|---------|--------|-------|
| Boyer-Moore for literals | ⚠️ | BC table built but unused (BUG-4) — degrades to first-byte filter + linear scan |
| Regex prefix extraction | ✅ | `Regex_GetPrefix` → BM on prefix → NFA verify |
| DFA compilation | ⚠️ | `Regex_DFAInit` works, but `Match_RegexDFA` has BUG-2 |
| Memory-mapped I/O | ❌ | Uses `SYS_READ` in 1MB chunks |
| Parallel file search | ❌ | Single-threaded |
| Early termination on binary | ✅ | Stops on first NUL byte |

---

## Error Handling

| Feature | Status | Notes |
|---------|--------|-------|
| File not found | ✅ | `"cannot open: "` error via `CCGrep_SendError` |
| Permission denied | ⚠️ | Silently skipped in recursive mode (like GNU grep). No error surfaced |
| Invalid regex | ❌ | `Regex_Compile` return value checked but no error message |
| Directory without `-r` | ❌ | BUG-1 — silently returns "(no matches)" instead of error |
| Result truncation | ✅ | `CCGrepState.truncated` flag in JSON response |

---

## IPC / Integration

| Feature | Status | Notes |
|---------|--------|-------|
| JSON-RPC-style protocol | ✅ | `{method, id, args}` → `{ok, content, truncated}` |
| Schema discovery | ✅ | Via `UtilArgs_ExportToolSchema` |
| Persistent connection | ✅ | One socket, reused across calls |
| 60s timeout | ✅ | Via `IPCDispatch.Dispatch` poll loop |
| Reconnect on desync | ✅ | `IPCDispatch_Reconnect` |
| Standalone testability | ✅ | Any JSON-speaking client can connect to `@halcode/Grep` |

---

## Summary Scorecard

| Category | Complete | Partial | Missing |
|----------|----------|---------|---------|
| Matching Engines | 6 | 1 | 0 |
| Output Control | 6 | 1 | 5 |
| File Selection | 4 | 3 | 4 |
| Regex Features | 3 | 0 | 2 |
| Performance | 2 | 2 | 2 |
| Error Handling | 3 | 1 | 2 |
| IPC/Integration | 6 | 0 | 0 |
| **TOTAL** | **30** | **8** | **15** |

**Overall: ~57% complete** (30 of 53 features). The core loop — find pattern, match line, emit with context — works solidly for single-file literal/regex search. The gaps are concentrated in:

1. **Output polish** (count-only, quiet mode, only-matching, color)
2. **File selection richness** (exclude globs, exclude-dir, symlink handling)
3. **Performance** (BM skip table integration, memory-mapped I/O)
