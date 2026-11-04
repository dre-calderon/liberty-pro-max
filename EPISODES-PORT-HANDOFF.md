# Liberty Pro Max: Episodes Port Handoff

Audience: a coding agent (GLM 5.3 Flash or similar) working a long-lived session.
Owner: Dre. Written by Shiv, 2026-11-04.

This document is the whole brief. It assumes zero prior context. Read it top to bottom before touching anything.

---

## 0. Read this first

**Mission:** get *The Lost and Damned* (TLAD) and *The Ballad of Gay Tony* (TBoGT) running natively on Apple Silicon through the `Awis13/liberty-pro-max` port, on top of the already-working base game.

**Definition of done:** from the port's launcher, selecting either episode boots that episode's campaign, renders, accepts input, and can save and reload a save. Both episodes, not one. "It compiles" is not done. "It gets to the loading screen" is not done.

**Status today:** the base game (GTA IV 1.0.8.0) runs. Episodes do not. There is **no loader** for an episode executable: no second lift path, no episode selection, no per-episode content or save plumbing. There *is* real episode-aware code already (section 3.1), because most of the fixes come from the FusionFix lineage, which shipped against EFLC. Do not mistake that for a working port.

**Hard rules, non-negotiable:**

1. Never fabricate progress. Every claim in a status report must cite a command output you actually saw, or a file you actually read.
2. Never run destructive git operations on the working tree (`git checkout -- .`, `git reset --hard`, `git clean -fd`) once port work has started. The depth-pipeline fix and the frame-limit feature described below are not committed anywhere yet.
3. Never launch or kill a game process against Dre's live install (`/Users/meatwad/Games/GTAIV`, `/Applications/Liberty Pro Max.app`). All testing happens against separate test copies. See section 10.
4. No public actions (PRs, issues, gists, pushes to upstream) without explicit approval from Dre.
5. If you are blocked, stop and write the blocker into the status file. Do not invent workarounds that hide the blocker.

---

## 1. The gate: you need `EFLC.exe` version 1.1.3.0

The port's entire design is "one Windows exe, lifted". The base game lifts `GTAIV.exe` 1.0.8.0. The episodes need the **standalone** episode executable, `EFLC.exe`, **file version 1.1.3.0**, plus matching episode data (`TLAD/`, `TBoGT/`, `common/` and the episode RPFs).

Things that do **not** work in its place:

- The Complete Edition `GTAIV.exe` 1.2.0.43. That build routes episodes internally through `TBoGT/setup2.xml` and `content.dat`, a mechanism 1.0.8.0 predates. It is not a substitute binary.
- The episode *data* already installed at `/Users/meatwad/Games/GTAIV`. That data is present and radio-patched, but there is no loader for it.

**Verification, run this first:**

```bash
ls -la /Users/meatwad/Games/ 2>/dev/null
find /Users/meatwad/Games ~/Downloads -maxdepth 3 -iname "EFLC.exe" 2>/dev/null
```

**If `EFLC.exe` is not found: stop.** Write one paragraph to the status file saying the gate is unmet, then report to Dre and wait. Do not start Phase 2+ work, do not write speculative episode code, do not "prepare" against a guessed binary layout. Everything in Phase 2 depends on addresses that only exist in that file.

Once the file is present, record its properties before anything else:

```bash
shasum -a 256 /path/to/EFLC.exe
python3 - <<'EOF'
import struct,sys
p="/path/to/EFLC.exe"
d=open(p,'rb').read()
pe=struct.unpack_from("<I",d,0x3c)[0]
print("machine",hex(struct.unpack_from("<H",d,pe+4)[0]))
print("image_base",hex(struct.unpack_from("<I",d,pe+0x34)[0]))
print("entry_rva",hex(struct.unpack_from("<I",d,pe+0x28)[0]))
EOF
```

Also confirm the file version string (1.1.3.0) from the PE resource block or from Windows, and write both the hash and the version into the status file. That hash becomes the new gate in `tools/lift.sh`.

---

## 2. Environment

| Item | Value |
|---|---|
| Host | MacBook, Apple Silicon M1 Max, 32 GB unified, macOS 27.0.1 |
| Free disk | ~440 GB at `/` |
| User | `meatwad` |
| Repo | `/Users/meatwad/dev/liberty-pro-max` |
| Game (base) | `/Users/meatwad/Games/GTAIV`, ~30 GB, `GTAIV.exe` 1.0.8.0 |
| Installed app | `/Applications/Liberty Pro Max.app`, version `0.3.3+depthfix` |
| Launcher defaults domain | `io.github.awis13.liberty-pro-max` |
| Game log | `~/Library/Logs/gtaiv.log` |
| Crash dumps | `~/Library/Application Support/gtaiv/crashes/*.txt` and `~/Library/Logs/DiagnosticReports/gtaiv-*.ips` |
| Shader cache | `~/Library/Caches/x86recomp/` |
| GitHub CLI account | `acalderon15` token active, git identity `dre-calderon` |

Toolchain state:

- Installed: Xcode CLT, CMake, Ninja, pkgconf (`brew install pkgconf` was required, `pkg-config` alone fails).
- **Not installed:** Ghidra, and no JDK on PATH. Both needed for Phase 2. Install:

```bash
brew install --cask ghidra          # 12.x, ~450 MB
brew install --cask temurin@21      # JDK 21, matches the author's JDK 21 setup
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
```

Port build trees already present:

- `out/build/portable` — the released/installed build (Ninja, Release, macOS 15 target, static deps).
- `out/build/release` — the author's default preset build; **this one carries `x86recomp/tools/lifter/recomp_lift`** which `tools/lift.sh` expects.
- `out/apple/prefix-macos` — static SDL3, FFmpeg, glslang, SPIRV-Cross built by `tools/apple_deps.sh`. Build time was about 25 minutes; it is already done, do not rebuild it.

---

## 3. Repo map

```
/Users/meatwad/dev/liberty-pro-max
├── install.sh                    build + install into /Applications
├── src/                          the port: hooks, fixes, launcher (25 files carry hardcoded guest addresses)
├── manifest/
│   ├── gtaiv.manifest            817 lines, lifter manifest, guest VAs in hex
│   └── native.list               397 hot functions, "<start> <last-byte-inclusive> <name>"
├── symbols/gtaiv/                exported from Ghidra analysis of GTAIV.exe 1.0.8.0
│   ├── functions.tsv             2.1 MB; columns: entry min max chunks thunk noreturn callconv name
│   ├── jumptables.tsv            switch tables, completed by tools/complete_jumptables.py
│   ├── imports.tsv, exports.tsv, pe_info.txt
├── tools/
│   ├── lift.sh                   the lift: exe + symbols + manifest -> generated/
│   ├── x86n.py                   per-function translator, called once per native.list entry
│   ├── ghidra_export.sh          Ghidra headless export (hardcoded Linux paths, needs editing)
│   ├── complete_jumptables.py    fills switch tables Ghidra leaves empty
│   ├── findpat.py                byte-pattern search in an image
│   ├── apple_deps.sh             builds the static dep prefix
│   └── local.env                 GTAIV_GAME_DIR=/Users/meatwad/Games/GTAIV
├── vendor/x86recomp/             SEPARATE GIT REPO, branch `liberty`, origin Awis13/x86recomp.
│                                 `vendor/` is gitignored in the top repo. Local fixes here are
│                                 NOT tracked by the parent repo and are NOT backed up anywhere.
└── generated/                    lift output (created by lift.sh; gitignored)
```

`manifest/gtaiv.manifest` line grammar (from `x86recomp/tools/lifter/main.cpp`):

```
entry <va> [name] | name <va> <name> | probe <va> [name]
resume <va> | setjmp <va> [name] | replace <va> [name] | hook <va> <end>
```

Base-game gate, `tools/lift.sh` line 18:

```
want=3d90e7c516fa450ca002e5031e62c0f66b404590f33e6cc9793b0da4fffbfd0f
```

That SHA is 1.0.8.0 and nothing else. The script refuses anything else, by design.

### 3.1 What already exists for the episodes (measured 2026-11-04)

The port is not episode-blind. It branches on the game's episode global, inherited from FusionFix, which shipped against the standalone EFLC binaries:

- **The episode global is guest VA `0x0112D754`** (`0` = GTA IV, `1` = TLAD, `2` = TBoGT). Referenced 9 times across 5 files: `src/gameplay_misc2.cpp`, `src/community_fixes.cpp`, `src/gameplay_misc.cpp`, `src/camera_controls.cpp`, `src/high_fps_fixes.cpp`.
- Example of a live episode branch: `src/community_fixes.cpp:472`, bike feet behaviour differs for episode 1 (TLAD).
- `src/camera_controls.cpp:122` treats episode 2 (TBoGT) as aim-zoom-follow.
- `src/high_fps_fixes.cpp:406` skips an IV-only cutscene fix outside episode 0.
- `src/update_folder.cpp:85` already knows the `TLAD\` and `TBoGT\` data folders and excludes them from IV's file scan.
- `src/launcher_options.h` already carries episode-shaped fixes (`HeliCeiling` "as in TBoGT", `TBoGTTracerFx`, `CompareStringFix` "TLAD calls").

**Consequences for the plan, and it matters a lot:**

1. That global is a VA in the **1.0.8.0** image. In `EFLC.exe` it sits somewhere else. Every one of those 9 reads has to point at the mapped address, or the port will think it is always playing GTA IV. This single global is the highest-leverage mapping in the whole job. Find it early, treat it as a named deliverable, and verify it at runtime by logging its value at the main menu of each episode, expecting 0, 1, 2.
2. Because the fixes already branch correctly, a large share of Phase 5's "the episode behaves like IV" bugs should not exist. What will exist: loader, lift, content paths, saves, and the mapped-episode-global failures.
3. Never assume the branch is right because the code looks right. Log the value.

### 3.2 The 230 hardcoded addresses

Measured in the working tree on 2026-11-04:

| File | Addresses | File | Addresses |
|---|---|---|---|
| `src/community_fixes.cpp` | 50 | `src/gameplay_misc.cpp` | 12 |
| `src/high_fps_fixes.cpp` | 34 | `src/fixes.cpp` | 12 |
| `src/widescreen.cpp` | 22 | `src/perf.cpp` | 12 |
| `src/threads.cpp` | 21 | `src/vehicles_misc.cpp` | 11 |
| `src/ff_shaders.cpp` | 19 | `src/camera_controls2.cpp` | 10 |
| `src/camera_controls.cpp` | 18 | `src/gameplay_misc2.cpp` | 8 |
| `src/shadows_lights.cpp` | 17 | `src/render_misc.cpp` | 7 |
| `src/ff_timecycext.cpp` | 17 | `src/options.cpp` | 7 |
| `src/update_folder.cpp` | 5 | `src/stability.cpp` | 4 |
| `src/jobs.cpp` | 5 | `src/bench.cpp` | 4 |
| `src/tree_shadows.cpp` | 2 | `src/coronas.cpp` | 2 |
| `src/render_misc2.cpp` | 1 | `src/integrity.cpp` | 1 |
| `src/diagnostics.cpp` | 1 | | |

**230 unique addresses across 25 files.** Reproduce the table with:

```bash
cd /Users/meatwad/dev/liberty-pro-max
grep -ohE "0x0[0-9A-Fa-f]{7}" src/*.cpp src/*.mm | sort -u | wc -l
grep -oE "0x0[0-9A-Fa-f]{7}" src/*.cpp src/*.mm | cut -d: -f1 | sort | uniq -c | sort -rn
```

---

## 4. How the lift works

```
EFLC.exe ──Ghidra──> symbols/eflc/{functions,jumptables}.tsv
     │                            │
     ├── manifest/eflc.manifest ──┤
     │                            ▼
     └────────────────────> recomp_lift ──> generated/*.cpp   (one big lifted image)
     │
     └── manifest/eflc.native.list ──> tools/x86n.py (parallel) ──> generated/native/*.inc
                                                │
                                     src/*.cpp + src/*.mm (port code, address-keyed)
                                                ▼
                                       cmake --build ──> app
```

The lifter does not need to understand the program. It needs three things pointing at the same image: the exe, the symbol export for that exe, and manifests whose VAs land on real function starts. Get those three out of sync and you get silent garbage at runtime, not a build error.

---

## 5. Phases

Each phase has an acceptance criterion. Do not start the next phase until the current one is met and recorded.

### Phase 0: fork, branch, protect the uncommitted work (30 min)

1. Fork both repos under `dre-calderon` (`gh repo fork`), add `upstream` remotes.
2. Commit the two uncommitted pieces in `/Users/meatwad/dev/liberty-pro-max/src/` (`launcher_mac.mm`, `launcher_options.h`, the frame-limit feature) on branch `fix-frame-limit`.
3. Clone/fork `x86recomp` properly and point `vendor/x86recomp` at the fork on branch `liberty`, then commit the depthless-shader fallback in `src/runtime/gfx/metal_device.mm`. That change currently exists only as an untracked local edit inside a gitignored directory. This is the single most fragile thing in the environment. Do it first.
4. Create branch `episodes` off `main` for the actual port work.

**Acceptance:** `git status` clean in both repos, both forks exist on GitHub, `git log --oneline` shows the two fix commits, `vendor/x86recomp` reports the fork as origin.

### Phase 1: correspondence tool and validation (half a day, no EFLC.exe needed)

Build `tools/addr_map.py`, described in section 6. Validate it on the base game first: map GTAIV.exe 1.0.8.0 onto itself and require 100% of the 397 `native.list` entries and 230 source addresses to resolve. That proves plumbing, not matching quality.

Then the real test, once FSOCIETY is reachable: pull the untouched Complete Edition `GTAIV.exe` **1.2.0.43** (17.4 MB) off the Windows PC, from `H:\GTAIVrepack` or a re-extract of the FitGirl archives. A different build of the same engine with a different compiler run is the honest proving ground. Target: match rate on the 397 native functions, with a written list of the ones that fail.

**Acceptance:** `python3 tools/addr_map.py --source <1.0.8.0> --target <1.2.0.43> --symbols ... --list manifest/native.list --out map.tsv` runs to completion, prints totals, and writes `map.tsv` with a confidence column. Numbers recorded in the status file.

### Phase 2: analysis and export of EFLC (needs the gate)

1. Edit `tools/ghidra_export.sh` for macOS paths, then export functions and jump tables from `EFLC.exe` into `symbols/eflc/`. Auto-analysis on a ~20 MB image with Ghidra 12: budget 30 to 60 minutes, run it headless, log to a file.
2. Run `tools/complete_jumptables.py` against `EFLC.exe` to fill switch tables.
3. Compare section layout to the base image (`symbols/gtaiv/pe_info.txt`): image base, `.text` extent, TLS directory, import DLL list. Note anything that differs, especially `xlive.dll` import count and TLS callbacks.

**Acceptance:** `symbols/eflc/functions.tsv` exists and is non-trivial (`wc -l` in the tens of thousands, same column layout as the base export), `pe_info` style summary written to the status file.

### Phase 3: mapping (the expensive, automatable part)

1. Run `addr_map.py` from the 1.0.8.0 image into the EFLC image for all 397 functions plus all 230 source addresses plus every VA in `manifest/gtaiv.manifest` (817 lines).
2. Produce three outputs: `map_high.tsv` (auto-accepted), `map_review.tsv` (ambiguous, needs a human read), `map_missing.tsv` (no candidate).
3. Manually review `map_review.tsv` and `map_missing.tsv`. Each entry is one function. Cross-check candidates in Ghidra when the signature is weak.

**Acceptance:** every one of the **1254 unique guest VAs** the port references (397 in `native.list`, 230 in `src/`, 1036 in `gtaiv.manifest`, after overlap) is either mapped with a stated basis, or listed as unmapped with a diagnosis. No silent gaps. Reproduce the count:

```bash
cd /Users/meatwad/dev/liberty-pro-max && python3 - <<'EOF'
import re,glob
nat={l.split()[0].upper() for l in open('manifest/native.list') if l.split()}
src={m[2:].upper() for f in glob.glob('src/*.cpp')+glob.glob('src/*.mm')
     for m in re.findall(r'0x0[0-9A-Fa-f]{7}', open(f,errors='ignore').read())}
man=set()
for line in open('manifest/gtaiv.manifest'):
    line=line.strip()
    if not line or line.startswith('#'): continue
    man|={t.upper() for t in line.split()[1:] if re.fullmatch(r'[0-9A-Fa-f]{8}',t)}
print("native",len(nat),"src",len(src),"manifest",len(man),"union",len(nat|src|man))
EOF
```

### Phase 4: lift and link EFLC

1. Write `manifest/eflc.manifest` and `manifest/eflc.native.list`, translated through the map.
2. Generate a per-episode lift path. `tools/lift.sh` is hardcoded to `GTAIV.exe` and one symbol directory; generalise it to take an exe + symbols + manifest set, keyed by an episode name, without breaking the base-game path.
3. Lift, translate the hot set with `x86n.py`, then build.
4. Expect build failures from the lifted code. Each is either a signature mismatch (fix the map) or a genuine translation gap. Do not paper over with stubs that return success.

**Acceptance:** the build completes (`BUILD=0`), the app links, and the episode path is selectable. A build is not a port.

### Phase 5: boot and fix loop, twice

For each episode: boot, read `~/Library/Logs/gtaiv.log`, fix the first fatal, repeat. Expect a long tail: missing HLE imports, RPF/asset path assumptions, episode-specific state, shader variants Metal rejects, threading, save-slot paths.

Episode plumbing to write. The fixes already branch on the episode global (section 3.1), so the missing pieces are structural:

- **The episode global.** Map `0x0112D754` into the EFLC image and repoint all 9 reads. Log its value at each episode's main menu and assert 1 for TLAD, 2 for TBoGT.
- Episode selection: launcher UI entry, persisted choice, and a flag reaching the game process.
- Startup path: which exe, symbol set, and manifest get lifted and loaded per episode.
- Content selection: which data roots and RPFs each episode reads, including `TLAD/` and `TBoGT/`.
- Saves: separate save directories per episode so TLAD, BoGT and IV never clobber each other.
- Boot validation per episode: a real frame rendered, input accepted, pause menu opens.

**Acceptance for done:** each episode booted from the launcher, played for 15 minutes, saved, quit, relaunched, loaded that save. Screenshots or log excerpts at `Application Support/gtaiv/crashes` times for both. Written into the status file.

---

## 6. `tools/addr_map.py` specification

Not present. Build it. It is the difference between 3 weeks and 3 months.

**Inputs:** source image path + its `functions.tsv`; target image path + its `functions.tsv`; a list of VAs (or native.list / manifest, parsed).

**Per VA, four stages:**

1. **Containment.** Find the function in the source `functions.tsv` whose `[min, max)` range holds the VA. Record offset from start.
2. **Byte signature.** Take N = 32 to 64 bytes from the function start (skip the first 0 to 16 bytes to dodge relocations and prologue variation), mask out every byte covered by a relocation in the source image, and hash. Build an index of the same masked hash over every function in the target image. Exact matches are accepted with high confidence.
3. **Fuzzy stage.** For unhashed candidates, normalise both bodies: strip rel32, rip-relative displacements and immediate operands to wildcards, then score by sequence similarity (difflib ratio is fine, or Rabin-Karp on opcode n-grams). Accept above a threshold, else push to review.
4. **Neighbourhood.** Build a call-graph of both images from the exported function boundaries plus call `E8`/`FF 15` scan. A function's callee set is often unique; use it to break ties. One hop is usually enough, two for anything stubborn.

**Outputs:** TSV `source_va<TAB>target_va<TAB>confidence<TAB>basis`. Plus a summary to stdout: totals mapped high, review, missing. Plus a `--report` markdown with the top 20 misses and why.

**Constraint:** the tool must never emit a guess as high confidence. A wrong mapping produces a port that boots and then corrupts state at random, which costs days to diagnose. Missing beats wrong.

---

## 7. Landmines, already-paid-for knowledge

These were all diagnosed the hard way today. Do not rediscover them.

1. **`vendor/x86recomp` is a separate repo on branch `liberty`, and `vendor/` is gitignored.** A top-level `git pull` does not update it. If you pull the parent and skip the vendor, the build fails with `no member named 'shader_tokens' in 'recomp::GameConfig'`. Update sequence:

```bash
cd /Users/meatwad/dev/liberty-pro-max
git pull --ff-only
git -C vendor/x86recomp fetch origin && git -C vendor/x86recomp checkout liberty && git -C vendor/x86recomp pull --ff-only
```

2. **SDL3 must be bundled by hand.** CMake links `@rpath/libSDL3.0.dylib` but never copies it into the app. Without it the app dies with `Library not loaded` off-machine. Copy `out/apple/prefix-macos/lib/libSDL3.0.dylib` into `Contents/Frameworks/`, `install_name_tool -id @rpath/libSDL3.0.dylib`, re-sign.

3. **`PerformanceOverlay` defaults to on and segfaults.** `MTL_HUD_ENABLED` leads to `SIGSEGV: host fault at 0x0` in `libMTLHud.dylib` at `-[HUDMTLLayerTracking safeAreaInsets]`. Must be off:

```bash
defaults write io.github.awis13.liberty-pro-max PerformanceOverlay -bool NO
```

4. **Depthless shader fallback (already written, uncommitted).** Three pixel shaders write `gl_FragDepth`; Metal rejects a pipeline that writes depth when the pass has no depth attachment, D3D9 tolerated it. The fallback rebuilds the MSL fragment function without the depth member and retries once. Log line to look for:

```
gfx: shader <hash> in a pass with depth format <n>: depthAttachmentPixelFormat is not valid and shader writes to depth; depth write dropped
```

The code is `DepthlessShader()` plus the retry in `MetalDevice::Pipeline` in `vendor/x86recomp/src/runtime/gfx/metal_device.mm`. **It is uncommitted.** Commit it in Phase 0.

5. **RESC10 out of video memory.** `commandline.txt` in the game folder carries `-norestrictions -availablevidmem 4096` (was 2047, which starved the streaming pool at 3600x2338). If it returns, lever order: in-game texture quality, then resolution, then raise `-availablevidmem`. Presently the game runs at 3600x2338 on a 120 Hz panel.

6. **Frame limit.** The runtime has an FPS limiter (`GTAIV_FPS_LIMIT`, `vendor/x86recomp/src/runtime/hle/present.cpp`, `PresentFrame`) that was wired only for iPad. It is now exposed in the Mac launcher as `FrameLimit` (0 display rate, 1 = 60, 2 = 40, 3 = 30), default set to 60. Uncommitted. Relevant because GTA IV vehicle physics and script timers misbehave at 120 fps.

7. **`launcher_options.h` LSP diagnostics lie.** The language server reports `No member named 'clamp' in namespace 'std'` and complaints about `NSInteger`. The file already used `std::clamp<NSInteger>` before these edits and compiles. Ignore the diagnostic, trust `cmake --build`.

8. **`CFBundleShortVersionString` is never bumped upstream** (still `0.1.0` on 0.3.3). Set it manually, and set `LSMinimumSystemVersion=15.0`, or the app lies about what it needs.

9. **`tools/ghidra_export.sh` is the author's machine**, hardcoded `/home/awis/labs/...`. Rewrite the paths for this Mac; do not run it as-is.

10. **`tools/run.sh` is also the author's Linux rig** (Xvnc, `/home/...`). Ignore it on macOS. Test by running the app bundle directly with `--game-dir <test copy>`.

---

## 8. Build and install commands

Portable build (what ships):

```bash
cd /Users/meatwad/dev/liberty-pro-max
export PKG_CONFIG_PATH=$PWD/out/apple/prefix-macos/lib/pkgconfig
cmake -S . -B out/build/portable -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=15.0 \
  -DCMAKE_PREFIX_PATH=$PWD/out/apple/prefix-macos \
  -DCMAKE_INSTALL_RPATH=@executable_path/Frameworks
cmake --build out/build/portable -j 8
```

Fast iteration on the renderer only:

```bash
cmake --build out/build/portable --target recomp_gfx -j 8
```

Install (only when Dre is not playing):

```bash
A="/Applications/Liberty Pro Max.app"
B="out/build/portable/Liberty Pro Max.app"
pgrep -f "MacOS/gtaiv" && echo "GAME RUNNING, STOP" && exit 1
cp "$B/Contents/MacOS/gtaiv" "$A/Contents/MacOS/gtaiv"
cp "$B/Contents/Frameworks/"*.dylib "$A/Contents/Frameworks/"
codesign --force --deep -s - "$A"
codesign -v --deep "$A"
```

Lift (base game, current behaviour):

```bash
cd /Users/meatwad/dev/liberty-pro-max
LIFTER=$PWD/out/build/release/x86recomp/tools/lifter/recomp_lift ./tools/lift.sh
```

`tools/local.env` already sets `GTAIV_GAME_DIR`.

---

## 9. Verification discipline

The failure mode of this project is not a crash. It is a confident wrong answer that boots, runs for 20 minutes, then corrupts state. So:

- After any build: `echo BUILD=$?` AND `grep -c "error:" <log>` AND confirm the artifact timestamp and size changed.
- After any install: `codesign -v --deep`, and `strings -a <binary> | grep <new feature string>` to prove the new code is actually in the bundle, not just built.
- After any lift: check `generated/` file count and that the new manifest VAs land inside `functions.tsv` ranges. A manifest VA that lands mid-instruction is a silent landmine.
- After any boot test: `grep -c FATAL ~/Library/Logs/gtaiv.log`, and confirm the window exists (`osascript` or `CGWindowListCopyWindowInfo`).
- Never report a phase complete off a tool's own self-report. Inspect the artifact.

---

## 10. Rules of engagement

**Testing never touches the live install.** Make a test copy once:

```bash
mkdir -p /Users/meatwad/Games/GTAIV-episodes-test
# hardlink the bulk, it is 30 GB and read-mostly
cp -cRp /Users/meatwad/Games/GTAIV/. /Users/meatwad/Games/GTAIV-episodes-test/
```

Run the port against the test copy with `--game-dir /Users/meatwad/Games/GTAIV-episodes-test`, from a copy of the app bundle in `~/Games/backups/`, not from `/Applications`. Timeouts on *your own* test process are fine and expected. Killing a process belonging to `/Applications/Liberty Pro Max.app` while Dre is playing is not.

- One writer per file. If parallel agents are used, files must be disjoint and integration happens with a real build after each merge.
- Commit messages: what changed and why, imperative, no marketing. Include the verification you ran.
- Never `git push --force` anything. Dre has a deny rule on one variant of it already; the intent covers the rest.
- Do not touch `/Users/meatwad/dev/VerticalFrame` or anything outside this project.
- Disk: keep at least 100 GB free. Repack copies and build trees add up fast.

---

## 11. Status file and reporting

Maintain `/Users/meatwad/dev/liberty-pro-max/EPISODES-STATUS.md` in the working tree, updated at every phase transition and every time you stop:

```markdown
# Episodes Port Status

Phase: <0-5> / <name>
State: <not started | in progress | blocked | done>
Last command run: <exact command>
Its output (tail): <verbatim>
Evidence artifact: <path on disk>
Next action: <one sentence>
Blockers: <none | precise blocker + what unblocks it>
```

Report to Dre in plain text when a phase closes, in this shape:

1. What was run.
2. What the output said, quoted.
3. What that means.
4. What is next.
5. What needs his decision, if anything.

No filler, no "I'm excited to", no summary of this document back to him. He already knows what the doc says.

---

## 12. Escalate to Dre, do not decide alone

- Buying or installing `EFLC.exe` and its data. That is his purchase and his machine.
- Any public GitHub action: issue, PR, push to upstream.
- Any change to `/Applications/Liberty Pro Max.app` while he is playing.
- Anything that deletes files outside `generated/`, `out/build/`, or your own scratch directories.
- Any decision to re-scope the goal ("TLAD only for now", "skip the radio", etc.).
- Any point where the honest match rate makes the remaining work look like weeks rather than hours. He needs the real number early, not a hopeful one.

---

## 13. Appendix: what "both DLCs working" implies

Sizing, so nobody is surprised:

- TLAD and TBoGT share the engine, so the second episode is materially cheaper than the first, provided the episode-selection plumbing is written generically rather than per-episode hacks.
- The bulk of the cost is Phase 3. **1254 unique VA references**, 25 files, one review pass each. Automatable to maybe 85 to 95 percent, the remainder is manual Ghidra reading.
- The long tail in Phase 5 is unknown by nature. Budget for it, do not promise through it.
- Prior estimate given to Dre: 200 to 400 hours of a competent agent. The mapper is what pulls it toward the low end.
