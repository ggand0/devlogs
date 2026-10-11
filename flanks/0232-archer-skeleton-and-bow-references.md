# The archer's skeleton and the bow references

Written by Claude Opus 5.5. 2026-10-11, handoff 103's "Before you can start" (Claude's two deliverables to the archer's Astra session), on Gota's order after the spearman's (devlog 0231). The game branch `feat/spearman-archer-clips` in `flanks-gfx` is unchanged. Everything is in `assets_dev` (main 618d583, 3cccee4, 0fa2532) and the untracked `refs/`.

## The bow had to bend

Handoff 103 planned a rigid stave, with each string half turning toward the nock. A joint only turns, and each half is 0.87 m. At full draw the hand is about 0.5 m behind the string's resting line, so each half would need about 1.0 m and stops about 14 cm short; the two halves don't meet. The old bow rig never had this problem: its limbs bend back and the string keeps its length (`bow_pose` in unit_pose.wgsl, `transforms` and `limb_angle` in tools/blender/archer/motion.py). The handoff left a bending stave for Gota's say. Gota chose bending limbs over a rigid stave with a stretching string. The stretching string would have added a scale channel to the clip format, the player and every tool.

## The skeleton

`assets_dev/_skel/archer.py`, `assets_dev/archer/skeleton_v1/` (README there). Twenty joints, the player's limit (`SKEL_JOINTS`):
- `pelvis`, `chest`, `head`;
- three per arm, with `arrow` under the draw hand;
- `limb_u` and `limb_l` under the bow hand, with `string_u` and `string_l` at the string's tips;
- three per leg.

Every joint sits at a pivot the model already carries for the old rig, so each rigid part rides its joint whole. The grip rides the bow hand, and only the knees and ankles blend. There are no coat joints (22 would be past the limit), so the tunic rides the pelvis as it does today.

The arrow needed one change to the model. It rests nocked on the bow, but a joint under the draw hand keeps its rest offset from the hand, so the arrow would have swung 0.73 m from the fingers. The build moves its 142 vertices (L0 only) to the nock in the draw hand, 1.5 cm below the hand's grip point, where the old draw holds it. The game hides the arrow outside a shot.

The far level is one part, skinned by the modeller's labels (`L3_surfaces.json`, which matches every vertex exactly). The label carrying moved from `spearman.py` to `glb.surface_labels`, and the spearman rebuilds byte-identical.

## The bow in `author.Pose`

- `stave(up, face)` turns the bow hand.
- `string()` reads the nock from the `arrow` joint, bends each limb by bisection until its tip is 0.87 m from the nock, turns each half to the nock, and lays the arrow 2 cm beside the grip, as `motion.py` did.
- `string(drawn=False)` braces the string.
- `blade()` now aims each kind's own weapon joint (`WEAPON`: `weapon` for the knight and spearman, `arrow` for the archer).

## Checks

- **The rest pose against the model:** 0.000 mm on every level.
- **The full-draw skin test:**
  - Both string halves end on the nock (off by 1e-15 m).
  - The grip is 0.72 m from the nock; `DRAW_LENGTH` in `motion.py` is 0.75.
  - The limbs bend 18.0° and 9.9°, because the nock sits above the grip's level.
- **Blender** reads 20 bones and both animations, and matches a linear-blend reference from the file to 0.0004 mm. Ours departs from linear blend by up to 20.3 mm at the bent knees.
- **The validator** now leaves the arrow out of width and reach and reports both with it. At rest he is 0.787 m wide and reaches 0.229 m ahead, or 1.04 m with the arrow in the hand.
- **The number check** compares our stave with M2TW's bow, both signed toward the upper tip. Against `peasant_archers:stand_a_idle:0` it comes out at RMS 15.6°, with three rows over: the bow 49° (M2TW tilts it forward; ours is upright, as the far levels' code pose), the feet (0.28 against 0.44 of the leg), and the draw forearm 17°.
- **The knight** is unchanged: the validator output on its seven clips and the number checks on four poses match the baselines byte for byte. The spearman is unchanged too.

## The bow references

`attack_missile_ready` (91 frames, 4.5 s), `attack_missile_hold` (9, 0.4 s) and `attack_missile_release` (29, 1.4 s) are in `m2clips.BASIC`, and only the bow types resolve them. They are rendered on the peasant archers into `refs/animation/m2tw_model_refs/peasant_archers/`. The bowman's skeleton type marks no impact frame for them, only `.evt` files.

## For the wiring, later

- The loader gives a model with bow rig parts the old bow rig, never the skeleton (`skinned` in `src/unit_glb.rs` needs the bow's clip rows empty).
- `SKEL_ROLES` asks for `weapon`, `coat_r` and `coat_l`.
- The arrow's visibility has to come from the clips' events.

All three belong to the archer's shot path in the player, which handoff 103 assigns to Claude.

## Open for Gota

The archer's carry: upright as the far levels draw it, or tilted forward 49° as M2TW's archer carries it. It can wait for Astra's ready pose.
