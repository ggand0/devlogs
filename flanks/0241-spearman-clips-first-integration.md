# The spearman's clips, first integration

Written by Claude Opus 5.5. 2026-10-13, about work of 2026-10-12, branch `feat/spearman-archer-clips` in the `flanks-gfx` tree, commits 7c2bc2e and 7fd69b9. Superseded in its clips by 4f4a3fa (devlog 0240); the code stays.

## 7c2bc2e: stand, ready and thrust

Handoff 109: the rear-grip model (`ready_v3/spearman.glb`, LFS), stand v1, ready v3 and thrust v7 as `spearman.stab.json`. The old jointed-arm attack table, which the far levels still play, moved to `spearman.arm_stab.json` (`attack_path`), since `<kind>.stab.json` is now the skeleton's stab clip. The model's scale is read from the trunk's height only, so the 0.7 m of spear above the head does not shrink him. The stab plays as the knight's does: wind-up to the clip's contact, follow-through to its end.

## 7fd69b9: shuffles carried from the knight's

A stopgap: the knight's four shuffles carried onto the spearman's ready pose (`assets_dev/spearman/shuffle_v1/build_shuffles.py`: the knight's root and lower-body turns in the world, ankles placed by the knight's foot paths, the upper body held). It passed the validator, but nobody looked at it frame by frame, and Gota found the legs awkward in play. Astra made the spearman's own shuffles (handoff 110, then 114); 4f4a3fa replaced the port and the ready and stab with them. The port folder stays on disk, uncommitted.
