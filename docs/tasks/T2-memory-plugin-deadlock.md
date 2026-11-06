# T2: memory-plugin deadlock

## Goal

- Fix the memory plugin's single-threaded deadlock: a held `pickup`
  socket blocks the agent daemon from parking its result, and agents
  die on timeout.
- This gates roadmap items 2 (auto-compaction), 3 (search) and 9
  (sub-agents), and streaming tool results. See
  `/home/bob/HalCode9000/docs/qol-roadmap.md`, "Architectural debt".

## Where

- Source: `/home/bob/halcode-plugins/cc_pgmem_ipc.ailang` (about 1,500
  lines). Design notes: `/home/bob/halcode-plugins/docs/DESIGN_PGMEM.md`.
- `halcode-plugins` is proprietary (`LICENSE-PROPRIETARY`). Don't copy
  its source into any public repo, and don't paste it into any external
  service.
- The HalCode trees also have a `cc_tools/cc_pgmem_ipc.ailang`. Find out
  how it relates to the plugin copy (same code, a stub or a fork), and
  report that.

## Do

1. **Reproduce first.** Write a test that starts the plugin, holds a
   `pickup` connection open, and shows a second client's `park` timing
   out. Record the timings.
2. **Find the cause.** Identify exactly where the plugin blocks: accept
   loop, per-connection read, database call, or result write.
3. **Fix it** with the smallest change that removes the block.
   - Preferred: an event loop that multiplexes connections with `poll`,
     plus a queue for results, so one slow call never blocks
     another client's store.
   - Avoid threads unless they are clearly simpler. The Haiku build has
     no threads yet, and thread joins have hung before.
4. **Keep the wire protocol unchanged,** so existing workers and agents
   don't need to change.
5. **Re-run the test:** the second client's store must complete while
   the first call is held. Also run any existing Pgmem tests, and a
   park/pickup round trip.

## Rules

- Don't replace `Ailang-Self-Hosting-/ailang.x`. Build with it into a
  scratch path.
- Don't commit or push. Don't touch the HalCode-Haiku tree; another bot
  is working in it.
- Don't print or log provider keys or database credentials.

## Output

- Report:
  - the cause, with file and line;
  - the change made, as a diff summary;
  - the test before and after, with timings;
  - existing tests and the round-trip result;
  - how `cc_tools/cc_pgmem_ipc.ailang` relates to the plugin copy.
