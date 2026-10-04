# A skeleton, one clip format, a clip player, a number check and a transfer tool

Written by Claude Fable 5.1.

2026-10-04. Gota found that every pose still took rounds of prompting Astra even with the M2TW model references (the knight's ready pose was at version five), asked for a way to speed the animation work up, approved the three steps of `docs/plans/022-animation-pipeline.md` and said to execute them. This entry is the record. Plan 022 has the design, the tables and the survey with its sources.

## Where it stands

| Piece | Where | State |
|---|---|---|
| Number check | `assets_dev/_m2tw_refs/pose_check.py` | works; committed |
| Skeleton, skin, clip library, authoring tools | `assets_dev/_skel/` | committed |
| Skinned knight, standing and ready clips, review sheets | `assets_dev/knight/skeleton_v1/` | committed; Gota has not yet looked at the coat and sleeve |
| Clip player | `flanks-gfx` tree, branch `feat/skeleton-clips`, 5b59b64 and 31a8db0 | runs behind `FL_CLIPS=1`; not pushed; perf run and play are Gota's |
| Transfer tool and its M2TW test | `assets_dev/_skel/transfer.py`, `_m2tw_refs/m2source.py`; output in `refs/animation/transfer_test/` | works |
| Survey of AI motion methods | plan 022 | done; the Kimodo trial waits for Gota |
| Brief for a new Astra session | `work/handoffs/093-knight-animation-workflow-for-astra-2026-10-04.md` | written |

`flanks-gfx` moved from `feat/knight-guard` to `feat/skeleton-clips` on Gota's word. `feat/knight-guard` had no commits of its own; backup ref `backup/knight-guard-2026-10-04` and bundle `work/backups/knight-guard-2026-10-04.bundle`. The 31 untracked scripts in `flanks-gfx/tools/blender/knight/` were removed on Gota's word after each was found byte-identical (SHA-256) to a snapshot committed in `assets_dev/knight/`.

## Step 1: the number check

`pose_check.py` reduces our pose and an M2TW pose to the same things and compares them: the direction of each upper arm, forearm, thigh and shin, the blade, the shield's face; the facing in plan of each foot, of the pelvis and of the chest; the chest's lean; three lengths as shares of each body's size. Ours come from the skeleton's forward kinematics; M2TW's from `m2anim.pose`, mirrored into our axes. The blade's axis is the long axis of the weapon's vertices, the shield's face the flat axis of the board's, both found by SVD on the mesh at rest and carried by the hand's or forearm's rotation.

On ready v5 against the Spanish militia's ready frame 0, root mean square 21.4 degrees, 11 of 18 rows over 12 degrees. The rows Astra fixed in v5 by reading the reference skeleton are near zero (foot facings 0.0 and 0.2 degrees, the shield-side shin 1.1, the blade 1.7). The large rows are real differences: the shield forearm 62.6 (ours is rigid), the chest lean 21.9 (10 against 32 degrees), the weapon arm about 22, the pelvis and chest facings (ours turn together by 55; M2TW turns the pelvis 39 and the chest 68), the pelvis height (0.91 of standing against 0.84, the shallower crouch Gota chose).

## Step 2a: the skeleton and the skin

Read Astra's `standing.py`, `ready_v2.py` to `ready_v5.py` and `joint_surface.py` first. Their deformation is, region by region, a blend between two transforms across a band: the elbow, knee and ankle blend rotations about the joint (formats 4 and 5), the sleeve root blends positions between the torso and the arm (format 3), the neck counter-turns the head. That is a skin with at most two joints per vertex, a parent and its child. So the weights are computed by the same rules from the rest positions, and one blend serves every region: the vertex turns about the child joint by the normalised blend of the two rotations.

Three things have no skeletal form in Astra's tables and changed:

- **The coat.** `skirt_scale` and `skirt_shift` are class constants per ready version. First try: coat vertices on the pelvis and the thigh of their side. It wrung the cloth, because the thighs' rotations in ready v5 are absolute and do not carry the pelvis's 55 degree yaw. Second try, kept: two helper joints, `coat_l` and `coat_r`, children of the pelvis at the hips, that take the thigh's swing from straight down as seen from the pelvis and none of its twist. The coat drapes over each thigh and splits on the centre line.
- **The shield arm.** The build script makes the two arms as mirrors, so the shield arm's elbow and grip are the weapon arm's mirrored. Its pieces are told apart by connected components: the sleeve is the piece nearest the shoulder, gloves and mail are pieces within 7.5 cm of the arm's line, the rest (board, straps, studs) rides `shield`. `shield` is a child of the forearm, as a strapped shield is and as M2TW binds its own.
- **L3.** One part, 52 triangles. The standing candidates had turned its shield in the mesh; the skinned model takes those 22 vertices back from the shipped model, so rest is rest on every level. It is skinned piece by piece.

`knight.from_astra` converts a 20-float standing row or a 49-float ready row into a sample. The skinned poses against Astra's own `deform`: torso and sword exact, head 0.09 mm, legs 0.15 mm, weapon arm 0.52 mm in ready v5. The differences are the three changes above plus the sleeve roots in standing v2 (26.9 mm; standing never anchored the sleeve root, ready has since format 3; one skin cannot do both).

The GLB writer (`_skel/glb.py`) appends to the file's buffer and keeps everything else byte for byte. Blender's glTF importer reads 19 bones and both animations, and its posed vertices match a linear-blend reference to 0.001 mm.

## Step 2b: the player

`FL_CLIPS=1`. Rust: `Skel` in the rig uniform (joints with parents, clip offsets, the joints the procedural layers write), `read_skeleton` in `unit_glb.rs`, the joint pair and weight in the pulled vertex's former padding, 46 pose slots for a skinned kind. WGSL: `put_skeleton` in the pose pass and a skinned branch at the top of `place`.

What it plays: standing and ready blended by the guard's readiness; the gait on the leg and coat joints; the chest's counter-turn and the arms' sway; today's sword blow and cheer on the arm joints, with the guard released during a blow so the body faces it. The whole-body steps in `place` (bob, lean, stagger, topple, facing) are untouched. The old path is untouched, and `sword_arm` is a copy of the sword branch of `put_rig_arm`, not a refactor of it, so nothing a knight does without the flag can have changed.

In play (arena attack, 2,000 knights, `tmp/shots/0195_clips/`): a regiment holding with the enemy near stands in the ready pose, shields forward; runners carry the sword up; the fight's rear ranks hold the guard. No shader error. `cmp_clips_vs_parts.png` puts the flag on above the flag off.

Checks: clippy with `-D warnings` clean, 35 tests pass. Fingerprints: the arena attack 21 of 21 equal between main, the branch with the flag off and with it on; the scripted 390k front (`FL_TEST_FRONT=1 FL_UNITS=200000`) 26 of 26 equal between main and the flag on. Main's binary came from `git archive main` unpacked in the scratchpad with the LFS files copied in, no worktree.

Not measured by the perf rules. The fingerprint runs logged 150 to 158 fps with the flag on and 140 to 157 on main at the default window with 390k soldiers, which shows no cliff and proves nothing more.

## Tools for authoring

- `author.py`: `Pose` builds a pose from limb directions, a blade direction with a roll, and ankle targets, the terms the number check prints. Rebuilding ready v5 from Astra's own numbers through it gives ready v5 with zero error on every joint. It holds each joint's turn relative to its parent, so turning the body carries the limbs. `tween` blends key poses.
- `validate_clip.py`: unit rotations, baked coat joints, ends equal to a given pose, no vertex under the ground, a foot down every frame, no joint faster than 1500 degrees per second. It reproduces Astra's numbers (ready v5: width 0.949324 m, reach 0.680175 m). A plain tween from standing to ready fails it with a foot 29 mm under the ground halfway, which is also what the player's joint blend does for a moment.
- `render_clip.py`, `clip_sheets.py`: our clips in the layout of the M2TW sheets. `clip_to_glb.py`: clips as glTF animations for Blender.

The handoff's command sequence was run as written, start to end.

## Step 3: transfer and survey

`transfer.py` works from bone directions, so limb lengths and rest poses do not have to match: pelvis and chest take the frames of the hips line and the shoulders line, limbs the source's directions through `Pose`, head and feet the source's own rotation from its rest, the blade its frame, the shield its facing with the roll it has on our forearm at rest. Two things went wrong on the way and are fixed: grounding by the ankle let a pitched boot's toe sink 10 cm, so the lowest point of either sole now touches the ground; and taking a round buckler's "long axis" for the shield's roll was arbitrary.

Test on the militia's ready, shuffle_left and cut_mid_right_to_left: 3.0, 1.7 and 1.7 degrees rms against the source over every frame, all three pass the validator. Sheets beside the M2TW ones in `refs/animation/transfer_test/` (local; M2TW's motion on our model, never for a repo).

The survey and its sources are in plan 022. Short form: NVIDIA's Kimodo (text plus pose constraints to a 30-joint skeleton, NPZ and BVH, licensed for commercial use, trained on licensed capture, runs locally) is the route an agent can run end to end; video capture of a person is the second route; research models and SMPL are out on license.

## Open, for Gota

1. Look at the coat and the sleeve roots: `assets_dev/knight/skeleton_v1/ready_review.png` and `stand_review.png`.
2. Play the branch with `FL_CLIPS=1`, and the perf run by the rules.
3. The Kimodo trial, about an hour.
4. The line in `AGENTS.md` about where Astra writes scripts: done the same day on Gota's word, in both the tree's `AGENTS.md` and `assets_dev/AGENTS.md` (assets_dev 29bc480).
5. A fresh Astra session with handoff 093, starting at the side shuffle.

## Traps met

- Blender's Python has no Pillow: textures are converted before Blender runs.
- A command chained with `&&` after the game check skips its copy step when a game is running; the later retry then lacks the file.
- `git archive` gives LFS pointer files: copy the real files in before building an export.
- `FL_DEMO=1` under `hashrun.sh` gave no fingerprints in 50 s (the sim had not started); `FL_TEST_FRONT=1 FL_UNITS=200000` runs on its own.
