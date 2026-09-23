# 0091: shield in the left hand, weapon arm rig (2026-09-23)

Branch `feat/knight-model`.

## Shield hand (d3b2af7)

Every soldier carried the shield in the right hand. The code-built meshes were drawn that way, and the loader mirrored anatomically authored models to match. The sim already credited shield cover on the left (movement.rs and arrows.rs `left_side`), so the fix is visual only. `unit_meshes::build_kind_lods` now mirrors the code-built meshes, the loader keeps models as authored, and three shader signs flip: the shieldwall turn, the shoulder twist and the sway. DIR and ARCHERY fingerprints 19 of 19.

## Weapon arm (c054238, ce9a7f6)

Gota: the stab slid the whole arm off the shoulder, and the spear was gripped at its middle. The stab added a translation to the rigid arm part, and the spear was welded to the arm.

- The weapon arm is a chain: shoulder, elbow, wrist. `arm_pose` solves two-bone IK from a grip goal and a weapon angle, and the vertex shader applies shoulder, elbow (blended over a band) and wrist turns. The hand never leaves the arm.
- The held weapon is its own part (id 8, `PART_WEAPON`). Rig data per kind (`unit_meshes::Rig`: shoulder, elbow, grip, tip direction, rear length, arm part, hold style) rides a uniform in each bucket's group 3, binding 6, in both render paths.
- Spear: upright at rest. Levelled by stance, watch range, charge or wall: upper arm hangs, forearm points ahead, shaft along it, and the spear slides forward through the hand so the grip ends near the butt (58% of the length behind the grip). The thrust draws back, then drives along the shaft.
- Sword stab: the hand pulls back, then thrusts along the blade with the wrist keeping the point near level. Classic swing, cheer and carry turn the whole chain about the shoulder.
- Archer draw hand (code-built only): pulls to the jaw by bending the elbow, no slide.
- Rigs: models carry them as the `weapon` part plus `pivot_weapon` and `joint_elbow` empties. Code-built kinds get theirs from builder constants. A model without them falls back to a rigid arm with no translation and a warning.
- Knight and spearman re-exported from the generators with `--reuse-bake`. Geometry, UVs, colours and atlas byte-identical to the approved files, only the part ids of 328 sword and 185 spear vertices and the two new empties differ. Knight generator changed in `tools/blender/knight/`. Spearman from a copy of Astra's scripts in `assets_dev/spearman/weapon_rebuild/`, Astra's files untouched. Note for Astra: tmp/drafts/handoff-weapon-rig-astra.md. The man-at-arms needs the same.
- Checked with an offline port (tmp/scripts/arm_view.py, side and front views of rest, level, draw and thrust) and in game in the charge test. Fingerprints 19 of 19, GPU check 0 of 13,800 frames.

## Open

- The shield arm is still one rigid part. The shieldwall turns it about the body axis and lifts it 0.1, which moves the shoulder. It wants the same chain.
- Thrust reach is small on Astra's arms: the rest hand is 0.40 m from the shoulder against 0.45 m of arm. The body lunge adds the rest.
- Upper body motion ideas from devlog 0088 still stand.
