# The spearman's skeleton and the spear stab references

Written by Claude Opus 5.5. 2026-10-11, for handoff 102's "Before you can start" (Claude's two deliverables to the spearman's Astra session). The game tree `flanks-gfx` is on the new branch `feat/spearman-archer-clips` from main a4c29b2, which Gota asked for after PR #27 merged. Nothing on it yet: all of this is in `assets_dev` (main 70a626e, d788c69, 44cd7f3) and the untracked `refs/`.

## What Astra's note 051 found

Astra's session started on the standing pose and stopped. Two reasons: `spearman/skeleton_v1/` did not exist, and the shared tools assumed the knight. `pose_check` built the knight's skeleton whatever model it was given, `Pose.blade` used the sword's frame (and divided by zero for an upright shaft), and the validator checked the coat joints only on a skeleton named `knight`. The number check also found no spear on the armored sergeants. All four were real.

## The skeleton

`assets_dev/_skel/spearman.py` and `assets_dev/spearman/skeleton_v1/` (README there). The knight's 19 joints in the knight's order, at the spearman's own pivots: `hand_r` at his wrist pivot, `weapon` at the grip. The shield arm has no pivots below the shoulder, so its elbow is the knight's, mirrored (the sleeve is the same tube in the geometry script), and its wrist is where the cuff ends.

The skin:

- **The spear arm** is three rigid parts with ball joints over the seams, as built in textured_v3. Each part rides its joint whole, which is also how the game's part pose turns them.
- **The shield arm** is one part. The knight's rule (pieces within 7.5 cm of the arm's line are arm) put the quilt sleeve on the shield at L0 and the hand on the shield at L1. Distances cannot separate these pieces: the hand grips the strap behind the board, and at L1 and L2 the grip strap is as near the arm's line as the sleeve is. The modeller's labels can. Each level's source mesh (`source_surfaces.json`, `L1_surfaces.json`, `L2_surfaces.json`, `L3_surfaces.json`, all tracked) labels every face, and every GLB vertex sits exactly on a source vertex at every level (0 mm). So each GLB triangle takes the label of the one face that holds its three corners, with no misses and no ambiguities. The `heater_*` pieces ride `shield`, and the rest of the part bends by the knight's arm rule.
- **Torso, neck, coat, legs:** the knight's rules, except that the spear arm's shoulder ball rides the chest by its label (its top reaches into the neck band).
- **L3:** by label: heater on `shield`, shaft on `weapon`, legs rigid on foot and thigh, the rest by the torso's rules.

Checks: the rest pose is 0.000 mm off the model on all four levels. Blender's glTF importer reads 19 bones, and its posed L0 matches a linear-blend reference from the file to 0.0005 mm, under the rest pose and under a hard-bent skin test. The skin test sheet shows clean bends. The knight's four shuffles keep a foot down and no vertex under the ground on this skeleton.

The rest pose is 0.998 m wide, against handoff 085's 0.9 m for standing. The number check against `armored_sergeants:stand_a_idle` puts the shield forearm 59° off: M2TW holds the kite across his body. By handoff 102, item 3, the standing pose is now Astra's.

## The tools

- `skeleton.bake_helpers` (moved from `knight.py`; `knight.bake_helpers` still works). `kinds.py` maps a skeleton's name to its module.
- `author.Pose.blade` reads the kind's rest frame and `BLADE_WIDTH`: the knight keeps the sword's flat upright, the spearman keeps the spearhead's width across him. A shaft pointed along that axis takes the smallest turn from rest instead of a zero vector.
- `validate_clip.py` checks the coat joints on every skeleton that has them.
- `pose_check.py` takes our skeleton and weights from the skinned model as stored. On M2TW's side, a weapon bound to the hand bone counts as held, which is how the sergeants' long spear is bound.
- `glb.add_skin` and `check_gltf.py` moved into `_skel` for every kind.

The knight is unchanged, checked against the old tools: the validator reports on all seven clips byte-identical, the number check on four poses byte-identical, the rebuilt `knight.glb` and its two clips byte-identical to the committed ones, and `Pose.blade` over 500 random poses identical (difference 0.0).

## The stab references

`m2clips.py` lists MTW2_Spear's centre stabs for the armored sergeants and the peasants: `stab_lo`, `stab_mid`, `stab_overhead`, `stab_mid_c`. Its skeleton type has only the eager_ slots. The parser now reads each clip's `-if` (the frame its blow lands: 11, 9, 24, 21), the sheet header names it, and the renderer always makes it a row of the sheet. The sword attacks get their frames too (18, 18, 13) the next time they are rendered. Sheets and GIFs: `refs/animation/m2tw_model_refs/{armored_sergeants,peasants}/stab_*`. Frames: `tmp/runs/m2tw_spear_stabs/`.

## For the wiring, later

- The skinned model stays out of `assets/units/` until Astra's ready pose exists: the player refuses a skinned kind without both `stand` and `ready` (`SKEL_CLIPS_NEEDED`).
- `SKEL_CLIPS` slot 7 loads `<kind>.stab.json`. For the spearman that name is today's legacy attack table, which is not a clip. The thrust has to take that file name when the table goes, or the slot has to move.
- The far levels' spear pitch (`spear = -1.42 * level` in `put_simple`) gets set from Astra's ready pose.

## Traps met

- `assets_dev`, `devlogs`, `work` and `tmp` in `flanks-gfx` are symlinks into the main tree. A script run from inside them sees the main tree's physical path. The knight's `build_skeleton.py` asserts the pre-skeleton `assets/units/knight.glb` (6952f4f1, bab8abb). Both trees now hold the skinned one, so the guard stops it. The regression rebuild took bab8abb's model from git.
- A game was running when the first renders were due; the Python checks ran while it played.
