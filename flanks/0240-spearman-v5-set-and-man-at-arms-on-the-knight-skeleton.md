# The spearman's v5 set, and the man-at-arms on the knight's skeleton

Written by Claude Opus 5.5. 2026-10-12, branch `feat/spearman-archer-clips` in the `flanks-gfx` tree, commits 4f4a3fa, 9971fe7 and dae0345 on c29ab30.

## The spearman (4f4a3fa)

Handoff 114: ready v5, thrust v8 as `spearman.stab.json`, shuffle_v4 forward and backward, shuffle_side_v1 left and right, copied byte for byte (the six hashes match). They replace ready v3, thrust v7 and my port of the knight's shuffles. The model and the standing pose already matched. No code change: the player reads each clip's rate (the stab is 90 Hz), its contact (0.333 s) and its travel (0.50 m and 0.40 m a play). The loader reports the clips; Gota stopped the in-game recording, so the motion in play is Gota's to see first.

## The man-at-arms (9971fe7)

He is built like the knight: the same parts (body, sword arm, legs, shield arm, sword) and pivots within 1 cm (elbow 4 mm, grip 10 mm). So:

- `assets_dev/man_at_arms/skeleton_v1/build_skeleton.py` puts the knight's 19 joints at his pivots (`knight.skeleton`) and weights his four levels by the knight's rules (`knight.skin`, L3 from L2 piece by piece). No overlaps in any band; the shield arm splits into 209 sleeve and 410 shield vertices at L0. The folder stays uncommitted in `assets_dev` until Gota approves it.
- All ten knight clips pass `validate_clip.py` on his skinned model: nothing under the ground, a foot down in every frame.
- The loader reads a kind's clips from another kind's files (`CLIP_KIND`): he plays `knight.<clip>.json`, so the knight's clips stay one copy.
- A cut clip plays the sim's slow cut, and a kind without one in the sim does not load it. He had none (his style 1 was the quick swing at 9 ticks), so in dae0345 he takes the knight's cut, 18 ticks to the blow and 21 back, by Gota's order: his two blows are the stab and the cut, both from the knight's clips at their own speed. `cut_weight` scales the cut's damage by its longer cycle, so his damage over time is unchanged; how often he hits first changes, since a cut gives the enemy longer to strike.
- His stab and charging blow play the knight's clips over his 9-tick wind-up (0.33 s against the knight's 0.43 s), so up to the contact they run about 1.3 times faster.
- One frame in play: he stands in the knight's standing pose, skinned on every visible man.
