# Knight charge pose v1

Written by GPT-6 Astra.

The updated handoff 104 narrows the next item to one held charge pose, followed by a separate charge strike. Created `assets_dev/knight/charge_v1/` on the private assets repository's main branch. The graphics branch and engine source are untouched. This candidate stops for pose review before the strike, as the handoff and AGENTS.md require.

## Implementation

`build_charge.py` begins with the canonical standing sample so root, pelvis and all leg channels remain exact neutral values. It authors chest yaw -32 degrees and lean 15 degrees; head yaw zero and lean 8 degrees. The right arm lifts the grip beside the head and the blade points forward/up/across. The left arm brings the shield ahead. The hand and sword share a global orientation so the existing grip stays attached. Shield forearm twist is solved by projecting the desired board normal around its fixed forearm axis, preserving the arm direction. The shield itself follows that forearm. Coat helpers are baked by the shared authoring tool.

`knight.charge.json` contains one sample, 79 floats, 19 parent-relative joint quaternions, 20 Hz, no loop, no travel and no event. It must be held over the procedural running legs. It is not a treadmill cycle. Duration is zero; player code must not divide by it. The later charge strike starts with this upper-body pose and returns to ready, with a separate lower-body transition to avoid snapping from the gait.

Source hashes in `source_checksums.json` cover the unchanged model and stand sample plus the current shared author, skeleton, knight, kinds and glb modules. These shared modules include Claude's uncommitted generalization for other kinds; they were read but not edited. Hashes were rechecked after rendering and remained equal.

## Reference work

Inspected full sheets and all ordered GIF frames for both dismounted_feudal_knights and swordsmen_militia: charge (13 frames), ready_to_charge (23), run_to_charge (19), charge_to_ready (31). Selected charge frame 6, 0.30 s, for one numeric correspondence before authoring. Only that pose was measured; no rotation curves or stride samples were transferred. All M2TW data, images, tables and overlays remain ignored under `local_reference/`.

Both arm chains, blade and chest match within 2.9 degrees. Eight of 18 comparison rows exceed tolerance against each unit. Seven are explicit neutral-lower-body differences: right foot facing 92.1 degrees, right shin 91.3, right thigh 26.4, left shin 21.1, left thigh 18.8, ankle stagger 32.8 percentage points and pelvis height 9.1 points. The gait will supply these in play; no stride was authored here. Shield face differs by 45.3 degrees against the knight and 53.8 against militia because of the fixed board mount. The retained board faces farther forward. Exact row explanations sit beside each local table.

## Validation and review

The shared validator passes: quaternion length error 5.78e-8, helper error 1.16e-7, zero ground penetration, both feet grounded. The saved sample clamps identically at negative, zero and positive times. Neutral pelvis and leg channels exactly equal standing.

All four levels have finite positions and normals, zero ground penetration and normal length error below 8.8e-8. Triangles: 2768 / 678 / 250 / 52. Widths: 1.189159 / 1.178325 / 1.169611 / 0.954430 m. L0 posed bounds: (-0.548874, 0, -0.292512) to (0.640285, 2.192122, 0.586247) m. The model remains 1.8 m tall in rest; the raised sword defines the greater posed height.

L0–L2 tests find zero blade/arm, weapon/body, shield/body, weapon/shield and leg/leg crossings. Coat/leg counts 68 / 104 / 72 exactly match standing; these are surface-crossing counts, not penetration depths. The existing sleeve elbow crease is visible. L3 omits arms and sword.

Checked charge, stand and ready neighbors in four directions at 1.4, 1.05, 1.0 and 0.9 m. All static cases clear at 1.4 and 1.05 m. Only the ready neighbor to the left crosses at tighter spacings. Running legs, gait extremes, blend transitions and approach travel have not been tested. No in-game run, sim edit or hash check was needed for this pose-only work.

A host-context process check found a running game, so Blender waited while CPU checks ran. After it exited, each background Blender launch was preceded by another host check. Rendered the saved clip through CPU skinning, not standard Blender skinning. The seven-view sheets use one 2.8 m camera scale across all levels; references use 2.2 m cells and the page states the difference. The orbit turns 360 degrees each way over 24 seconds at 24 fps, with eased reversal. It shows a fixed pose, not authored leg motion.

Reviewed all level sheets, complete reference sheets and ordered reference frames, plus the 54 timed browser captures covering both orbit directions and the loop. Full MP4 decoding and browser time/frame advancement passed. This establishes the static pose's framing and visible surfaces, not how its blend over a live gait looks. Review outputs and validation reports are in the iteration folder.

## Delivery

Integration note: `work/notes/062-knight-charge-pose-v1-2026-10-11.md`. Review: `assets_dev/knight/charge_v1/review.html`. Added its iteration row while preserving the other pending README changes.

Clip SHA256: `0700e92e8fc549a6b67ed930e5755da74e6a19816d4f9b24b58224291bd9d908`.

The automatic approval reviewer rejected the staging command because it interpreted the AGENTS.md sentence as forbidding individual-file staging. The command did not execute; no files were staged or committed. The candidate is complete on disk, with archive commit pending clarification. No push was attempted.

Next: Gota reviews this pose, then create charge_attack after Claude's reference preparation is available. The strike uses 0.4 s to contact and 0.6 s to ready. A full charge run cycle remains deferred unless the existing gait looks wrong in play; overhead remains optional and unstarted.
