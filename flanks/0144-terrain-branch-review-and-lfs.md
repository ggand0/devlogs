# Terrain branch review, LFS migration and the big-binary rule
Written by Claude Fable 5.1.

Date: 2026-09-26, night. Branch `feat/terrain-look` in flanks-gfx, reviewed from the main tree before its PR. `main` unchanged at 66eb416.

## Review

Seven commits, all authored as Gota, no Codex footer, no co-author line, present-tense titles. Every added line of code, shader, scripts, license note and JSON scanned for AI tells, slop words, "owner", "the user" and em dashes: none. Four comments use a semicolon, harmless. The diff touches only terrain.rs, terrain.wgsl, camera.rs and assets/terrain, so no shared-file claims, and a test merge onto today's main has no conflicts. The classic heights changed on purpose (Astra's devlog 0142), so every fingerprint changes at the merge and the gate baselines get regenerated then. Gota has played the map.

## What the review caught

The branch added 30.5 MB of terrain textures in plain git. The repo tracks only `assets/units/*.glb` with LFS, and the repo is public, so each texture and every later version of it would have sat in history forever. Gota: use LFS, and write the rule into both AGENTS.md files.

## Rewrite

The branch was unpushed, so its history was rewritten in place. Backup first: ref `backup/terrain-look-pre-lfs-20260926` and a verified bundle at `~/ggando/gamedev/flanks-backups/terrain-look-pre-lfs-2026-09-26.bundle`.

1. Squash. Gota wanted the Grass 004 trial gone if it could go cleanly. Its commit and the restore that followed net to four text files (tile scale 10 m to 2.51 m, per-layer normal strength, a `--layers` option in the packing script, the tile size in the notes) and no binary change, so the pair became one commit, "Use the pasture at its natural scale". Built with `git commit-tree` from the original trees, authors and dates, so no working tree was touched and the new tip's tree is identical to the old tip's. The Grass 004 blobs no longer exist in history.
2. LFS. `git lfs migrate import --include="assets/terrain/*.ktx2,assets/terrain/*.png" --include-ref=refs/heads/feat/terrain-look --exclude-ref=refs/heads/main --yes`, run in flanks-gfx where the branch is checked out. Six commits rewritten, `.gitattributes` gains the two patterns in every commit, each texture blob is a 132-byte pointer, main untouched. Trap: migrate's own checkout left the working files as pointers; `git lfs checkout` restored them from the local objects, and all nine files match their pre-migration sha256.

Hash map, old to new: 42e30de c0eb958, c9303f3 b94e30c, fec2ff6 55b011c, 064e0fc + 0e526cb f299513, e0b06c7 6806f60, 2adeb91 b245781 (the accepted baseline). Recorded in the terrain handoff for Astra.

## Rule

Both AGENTS.md files, under "Where files go": big binaries go through Git LFS, pattern in `.gitattributes` before the first commit, check with `git lfs ls-files`. The assets_dev copy is committed there (0aa2dbd). At every branch review, a `Bin` line in the diff stat whose path has no LFS pattern is a finding.

## Answers recorded

Terrain performance: about 0.4 ms of GPU for the whole opaque pass at 200k on a CPU-bound frame, nothing on the counter; Gota's fps read is the check. The boundary-blend precompute Astra left to Claude stays deferred: at most 0.09 ms of GPU by Astra's measurement, the corridor is specific to this layout artwork, and the larger authored map will get explicit surface masks that replace the blend. A bigger map does not make the blend costlier, since it is per pixel on screen, not per map area. Vegetation is the next graphics branch, off main after the terrain merge.
