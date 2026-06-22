---
name: ailang-dev
description: Writing, editing, and compiling AILang source files. Use when working on .ailang files, debugging compiler errors, or building HalCode9000 and its cc_tools.
---

# AILang Development

## Compiler

```bash
# Installed to PATH — works from any directory
ailang.x path/to/source.ailang path/to/output.x
analyzer.x path/to/source.ailang          # syntax check without compiling
```

Build everything via `build.sh` from AILangSH root:
```bash
# From /mnt/c/Users/Sean/Documents/AILangSH/
./build.sh              # rebuild HalCode9000 + all cc_tools
./build.sh --no-tools   # main binary only (fast iteration)
./build.sh --tools-only # cc_tools only
```

## Language Rules

- **6-arg syscall limit** — SysV AMD64 uses RDI/RSI/RDX/RCX/R8/R9. analyzer.x enforces this.
- **StoreValue** defaults to 8-byte (qword). Use `StoreValue(addr, val, "dword")` for 4-byte writes.
- **No trailing `\n` in string literals** — AILang supports `\n` mid-string (e.g. `"line1\nline2"`) but NOT as the very last escaped character: `"text\n"` is a syntax error. Use a literal newline between the last character and the closing `"` instead. This is the ONLY escape-sequence restriction.
- **MemoryCopy/MemorySet** emit CLD + REP MOVSB/STOSB with register save/restore.

### String Literals

AILang **has string literals**: `"hello"`, `"error: %d\n"`. The compiler packs them as length-prefixed byte sequences in the data section with automatic null termination.

- **No char literals** — `'x'` is a syntax error. For single bytes, use the integer ASCII code (e.g., `72` for `'H'`).
- **Byte-by-byte construction** — when you need dynamic strings, use `Allocate` + `SetByte` to build them one byte at a time, then null-terminate:
  ```
  buf = Allocate(256)
  SetByte(buf, 72)     // 'H'
  SetByte(Add(buf, 1), 101)  // 'e'
  SetByte(Add(buf, 2), 0)    // null terminator
  ```
- **ASCII-code scanning** — to detect patterns in raw bytes, compare against integer ASCII values: `H=72 O=79 M=77 E=69 ==61`. This is how `Auth_GetHome` finds `HOME=` in `/proc/self/environ`.
- **StringConcat** (from lib-stringutils) joins two null-terminated strings into a new allocation.
- **Trailing `\n` rule** (see above) is the only escape-sequence restriction — all other escapes work normally.

## Common Patterns

### Function definition
```
Function.MyFunc {
    Input: param1: Address
    Input: param2: Integer
    Output: Address
    Body: {
        ReturnValue(result)
    }
}
```

### IPC tool structure
Every cc_*_ipc tool follows: `FixedPool` constants → `BuildSchema` → `DoWork` → `SendError/SendResult/SendSchema` → `HandleRequest` → `Main` (bind socket, accept loop).

### Reading HOME from environment
Read `/proc/self/environ`, scan for `H=72 O=79 M=77 E=69 ==61` to extract `$HOME`. See `Auth.ailang:Auth_GetHome` for the reference implementation.

### Fork/exec subprocess
```
pf = Allocate(8)
SystemCall(SYS_PIPE, pf, 0, 0, 0, 0)
pipe_r = Dereference(pf, "dword")
pipe_w = Dereference(Add(pf, 4), "dword")
argv = Allocate(32)
StoreValue(Add(argv, 0), "/bin/sh")
StoreValue(Add(argv, 8), "-c")
StoreValue(Add(argv, 16), cmd)
StoreValue(Add(argv, 24), 0)
pid = SystemCall(SYS_FORK, 0, 0, 0, 0, 0)
// child: dup2 pipe_w->1, execve, Exit(127)
// parent: close pipe_w, read pipe_r, wait4
```

## cc_tools Syscall Numbers (x86-64 Linux)
| Name | Number |
|------|--------|
| SYS_READ | 0 |
| SYS_OPEN | 2 |
| SYS_CLOSE | 3 |
| SYS_PIPE | 22 |
| SYS_DUP2 | 33 |
| SYS_FORK | 57 |
| SYS_EXECVE | 59 |
| SYS_WAIT4 | 61 |
| SYS_MKDIR | 83 |
| SYS_UNLINK | 87 |
| SYS_READLINK | 89 |

## WSL2 Rules (always apply)
1. Never `find /`, `/mnt`, or `/mnt/c` — hangs permanently on NTFS.
2. Always scope `find` to a specific subdirectory.
3. Pipe all unbounded output through `head`/`grep`/`tail`.
4. Abstract sockets (`@halcode/*`) bypass WSL2 tmpfs — use them, not `/tmp/*.sock`.
