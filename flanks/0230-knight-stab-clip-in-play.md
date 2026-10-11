# The knight's stab clip in play

Written by Claude Fable 5.1. 2026-10-11, branch `feat/skeleton-clips` in the `flanks-gfx` tree, on c8bd74c. Astra's handoff: `work/handoffs/105-knight-stab-integration-for-claude-2026-10-11.md`; the approved clip `assets_dev/knight/stab_v1/knight.stab.json` (SHA-256 d4aa62e4..., 21 samples at 20 Hz, contact at 0.40 s, 1.00 s long, zero travel, the ready pose at both ends).

## What changed

- `assets/units/knight.stab.json`: the clip, byte for byte. The legacy `<kind>.stab.json` attack tables share the suffix, but only a model with a jointed arm (the spearman) reads one (`arm_rig`); the knight's loader takes the skeleton path, so the two never meet.
- The clip slots grow from 8 to 12 (`SKEL_CLIP_SLOTS`, the two arrays of `Skel` in unit_pose.wgsl), for the stab now and the charge, the charge blow and the overhead to come. `SKEL_CLIPS` gains `stab`, slot 7 (`CLIP_STAB`).
- A near knight's ordinary stab (style 0, no charge bit, no cheer) plays the clip exactly as the cut plays its own: the sim's wind-up maps onto the clip before its contact and the follow-through onto the rest, so at 12 ticks and `FOLLOW_S` 0.6 it runs at the speed it was made. The code's sword-arm stab no longer plays on the joints for it; the blow lean is dropped as for the cut; on the move the legs keep the gait, as for the cut, and the steps blend under it. The charging blow stays the code's, and the far levels play the code's stab by parts as main does.
- `pre.stab` beside `pre.cut`; `put_skeleton` picks the clip (`blow_clip`) and treats both the same.

## Checks

opt-dev build, strict clippy and the 36 tests pass. The arena runs with no error lines and the loader reports the eight clips. Not played by me; Gota plays it: a complete stab and its return into the guard, the alternation with the cut, a stab on the move, and the L1/L2 line.

## The branch closes

Gota: "is it feature complete now? where's the pr draft". Yes for the knight. The one open item of handoff 095, the promotion of the accepted clips' authoring scripts into `tools/blender/knight/`, was done and then undone the same evening on Gota's call: a build script per clip per kind in the public repo is noise that does not scale ("are you gonna copy the same shit for ALL the animations over the future?"). The public repo ships the clips; the scripts stay with their iterations in `assets_dev`, which already keeps them with their checksums. The rule in both `AGENTS.md` files still says to copy the scripts on approval and should change to match; that is Gota's edit. The stab integration was recommitted without the tools files (e71008a) and the scripts commit dropped with `git reset`.

PR draft: `work/drafts/046-pr-skeleton-clips.md`, with its plain rewrite beside it (`claudish-md`, docs/internal/007). Open after the merge: the charge cycle and the charge blow, an overhead clip (optional), the spearman and the archer, and the sameness of the guards in a press (memory `guard-variety-in-a-press`).

## Gota's question of the day

Mixing knights in the standing stance during a fight so the old code animations could add variety: answered no in the thread. In a press a man standing straight among crouched guards reads as not fighting, and the code's arm motions over a side-on body were exactly what looked wrong. Variety comes from more clips on the same guard (the cut, the stab, the overhead to come), and later from idle variants authored as clips.
