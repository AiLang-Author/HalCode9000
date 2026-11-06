# T1: shared-core merge map (step 0, analysis only)

## Goal

- Produce the merge map for step 0 of `docs/qol-roadmap.md`: exactly
  what differs between the four HalCode code bases, and which version
  of each difference the shared core should keep.
- **This task is analysis only. It changes no source file.** The merge
  itself is a later task that works from this map.

## Trees

- `/home/bob/HalCode9000`: the canonical core, terminal UI.
- `/home/bob/HalCodeGTK`: desk + GTK shell.
- `/home/bob/HalCode-Haiku`: desk + Haiku window.
  - Its committed tree (`git show HEAD:<file>`) equals HalCodeGTK.
  - Its working tree has uncommitted Haiku platform work in `Platform/`
    and the files that use it. Another bot is editing this tree right
    now, so read it but never write to it.
- `/home/bob/HalBot`: the bot. It also carries `HalCode9000.ailang`,
  `cc_tools`, `backends` and the other shared files.

## Do

1. List every file that exists in more than one tree, and mark each
   one as:
   - **shared core:** agent loop, `IPCDispatch`, `History`, `Auth`,
     `backends/`, `providers/`, `cc_tools/`, `Platform/`;
   - **front end:** `UI.ailang`, `shell/`, desk, window;
   - **unclear:** say why.
2. For each shared-core file, diff the trees and classify each
   difference:
   - a feature or fix present in one tree only: name it, and say which
     tree has it;
   - a conflicting change to the same code: show both sides;
   - a front-end or platform difference sitting in a shared file:
     propose moving it behind `Platform/` or into the front end.
3. For `HalCode9000.ailang`, list the functions unique to each tree.
   Known so far: GTK and Haiku have the host-mode functions
   (`CC_HostCtlOpen`, `CC_HostShutdown`, `CC_HostArmSig`),
   `CC_StatusReport`, `CC_ParkDigest`, `CC_FormatSessionList`,
   `CC_ResolveWorkerSpec`, `CC_ProvFromJSON`, `CC_HttpStatusHint`,
   `CC_ReapChildren` and more. HalCode9000 has newer tool fixes, such
   as `find -L` in Skills. Complete the list for all four trees.
4. For each of the 13 `cc_tools` workers that differ between HalCode9000
   and GTK, state what differs and which version to keep.
5. Propose a sync script: which paths are copied from HalCode9000 into
   each other tree, and which are never touched there. Write the
   proposal; don't create the script.

## Rules

- Read only. Don't edit, build, commit or push anything.
- Don't print the contents of `~/.halcode`, key files or provider keys.

## Output

- Write `/home/bob/HalCode9000/docs/tasks/T1-merge-map-result.md` with:
  - the file classification table;
  - the per-file difference list, each with a keep decision or an open
    question;
  - the proposed sync script paths;
  - a count of open questions that need the owner.
