Read this first, all of it, before you run anything:

/Users/meatwad/dev/liberty-pro-max/EPISODES-PORT-HANDOFF.md

That document is the complete brief. It assumes you know nothing about this project. Follow it literally, including the hard rules in section 0 and the testing rules in section 10.

The job: port The Lost and Damned and The Ballad of Gay Tony to the `liberty-pro-max` GTA IV port on Apple Silicon (macOS 27, M1 Max). The base game already runs. The episodes do not exist yet.

Your first action, before anything else, is Phase 0. That is the fragile part: `vendor/x86recomp` is a separate gitignored repo holding two fixes that exist nowhere else, one of which is the depthless-shader fallback that currently keeps the game from crashing. Get both repos forked under `dre-calderon`, branch the fixes, commit them, and only then touch anything else. Details in the doc, section 5, Phase 0.

Then Phase 1: build `tools/addr_map.py` per the spec in section 6 and validate it on the base game (1.0.8.0 mapped onto itself, expect 100 percent of the 397 native.list entries to resolve). Spec is explicit; do not invent a different approach without saying why.

Phase 2 is gated. It needs `EFLC.exe` version 1.1.3.0, which is not on this machine yet. Verify that with the command in section 1. If it is missing, do not start Phase 2, do not write speculative episode code, do not "prepare" against a guessed layout. Finish Phase 0 and 1, write the status file, and report.

Rules that matter more than progress:

- Never fabricate a result. Every claim cites command output you actually saw or a file you actually read.
- Never launch or kill a process against `/Applications/Liberty Pro Max.app` or `/Users/meatwad/Games/GTAIV`. Dre plays on those. Test against copies, per section 10.
- No public GitHub actions. No PRs, no issues, no pushes to upstream, without Dre saying so.
- No destructive git: no `git reset --hard`, `git checkout -- .`, `git clean -fd`. There is uncommitted work in the tree right now.
- Do not touch anything outside `/Users/meatwad/dev/` for this project.

Maintain `/Users/meatwad/dev/liberty-pro-max/EPISODES-STATUS.md` in the shape given in section 11, and update it whenever you stop. When you report back, give Dre: what you ran, what the output said quoted verbatim, what it means, what is next, and any decision you need from him. No filler, no summary of the brief back at him.

Start with Phase 0. Tell me when it is committed and verified, or where it broke.
