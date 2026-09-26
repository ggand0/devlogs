# 0136: Two working trees, one game at a time

Written by Claude Fable 5.1. 2026-09-26.

Gota wants Claude on core game and performance work and GPT-6 Astra on graphics branches at the same time, on one machine. Opus 5.5 drafted the plan (docs/plans/016-parallel-workspaces.md); this session reviewed it, simplified one part, and set it up.

## What was decided

- A git worktree, not a second clone. `flanks-gfx` sits beside `flanks`, shares the repository, and gets the ignored folders (devlogs, work, tmp, docs, assets_dev, resources, data, the two agent rule files) as symlinks into the main tree. The main tree keeps its name and path: Claude's memory, session history, the scripts and the other linked tree are keyed to it.
- One game at a time is a process check, not a lock. The plan had a `flock` scheme with a cargo runner and an environment variable. Gota wanted something simpler, and a process check tests the real invariant: no second flanks process, including one he started by hand, which a lock between cooperating launchers can never see. `work/scripts/flanks-run.sh` refuses to start if any process named `flanks` or `flanks-<name>` exists, checks again one second after the start and backs off if an older game exists, and runs the game under `exec` so a killed wrapper takes the game with it. With no arguments it lists running games. Tested with fake processes: refuse, back-off, plain run, list.
- Builds and Blender count as well: the rule in CLAUDE.md and AGENTS.md is to run the wrapper's list mode before a launch, a build or a render, and wait while it lists anything. Never kill a game you did not start.
- Fingerprints are not the gate for graphics changes. The diff is; Claude reviews it before merge. The hash run is for changes that touch what the sim reads (heights, water, a shared random source) or for proving an `FL_` switch off is unchanged. My first draft of that rule was compressed to the point of being unreadable; Gota asked for plain sentences and got them.
- AGENTS.md now grants Astra the graphics-owned files (`terrain.rs`, `vegetation.rs`, `water.rs`, `camera.rs`, `assets/shaders/`, `assets/terrain/`) on graphics branches in its tree without asking; everything else in `src/` stays ask-first.

## What was done

- `work/scripts/`: the five scripts with a hard-coded `cd` into the main tree now take the tree from `FL_TREE` (default: the main tree). Through the `work/` symlink the scripts are the same files in both trees, so without this Astra's fingerprint or screenshot runs would have used Claude's binary silently. The asset path itself is safe: main.rs bakes it in from the crate directory at build time. The seven launching scripts (`hashrun`, `gate`, `clip`, `shots`, `gpu-pass-measure`, `pile-wide`, `pile-two`, `perf/bench4`) start the game through the wrapper. `clip.sh` and `shots.sh` lost their `pkill -x flanks`, which would have killed a game they did not start.
- Check: a 25 s `FL_TEST_DIR=1` hash run from the main tree through the patched script gave 11 fingerprints, all equal to `work/baselines/dir-footwork-1dd582f.hash`. From the graphics tree with `FL_TREE` set the script looks for the graphics tree's binary (none built yet); with `FL_TREE` unset it runs the main tree's.
- The worktree: `feat/terrain-look` at fde9a0c, ten symlinks, exclude entries added to the shared `.git/info/exclude` without trailing slashes (the `.gitignore` patterns end in `/` and would let symlinks through). `git status` is clean in both trees.
- The Astra brief (work/notes/astra-terrain-look-2026-09-25.md) points at the real tree, `FL_TREE`, and the wrapper instead of the old marker file. `work/notes/board.md` started: one line per tree with branch, claimed shared files, and what is running.
- docs/internal/003-parallel-worktrees.md: the layout, what is shared, the rules, typical confusions, how to recreate and remove the tree. Meant to be pasted into a session that gets confused.
- CLAUDE.md, AGENTS.md and assets_dev/AGENTS.md: every devlog, internal doc, handoff and note starts with a line naming the model that wrote it.
- Backups of the cleaned-up project (everything but `target/` and the disposable `tmp/`) to /data/ggando/flanks and /data_hdd2/ggando/gamedev/flanks, checksum-verified. The nightly devlog sync to /data/devlogs/flanks was already in place.

## Next

Gota or Astra builds the graphics tree (`cd ../flanks-gfx && nice -n 19 cargo build --profile opt-dev -j 8`, not while a game runs), then Astra starts there with the terrain brief and Claude cuts the perf branch in the main tree.
