# T4: analyzer.x usage failure on Linux

## Goal

- On the Haiku guest, `analyzer.x` printed its usage message and
  exited 1 for valid input. Find out whether the same happens on Linux,
  and why.
- This decides whether roadmap item 7 (auto-verify) is blocked on every
  version, or only on Haiku.

## Where

- Analyzer source and build: find them in
  `/home/bob/Ailang-Self-Hosting-`. Start from where `analyzer.x` is
  built, and from the `cc_ailang_ipc` worker that calls it
  (`/home/bob/HalCode9000/cc_tools/cc_ailang_ipc.ailang`).
- The guest copy was `/boot/home/analyzer.x` (428,362 bytes, built
  2026-11-04).

## Do

1. On Linux:
   - build `analyzer.x` from the current tree into a scratch path, with
     the trusted `./ailang.x`;
   - run it the way `cc_ailang_ipc` runs it, on two or three valid
     `.ailang` files;
   - record the exit code and output.
2. If it fails on Linux, find the cause: argument parsing, reading
   `/proc/self/cmdline`, file open, or something else. Fix it if the fix
   is small and local, and explain it.
3. If it works on Linux, compare with how it runs on Haiku:
   - the guest copy is a Linux ELF, so it runs through the bridge;
   - the arguments reach it through the bridge's `/proc/self/cmdline`.
   Report the likely Haiku-only cause. Don't run anything on the Haiku
   guest; another bot is using it.

## Rules

- Don't replace `Ailang-Self-Hosting-/ailang.x`. Don't commit or push.
- Don't run or touch the Haiku VM.

## Output

- Report:
  - whether it fails on Linux: yes or no, with the exact command, exit
    code and output;
  - the cause;
  - any fix made;
  - what that means for auto-verify on each version.
