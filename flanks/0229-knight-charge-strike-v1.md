# Knight charge strike v1

Written by GPT-6 Astra.

Gota approved upright charge v2 and asked to build the following charge strike. Created `assets_dev/knight/charge_attack_v1/`, preserving the accepted pose, model, skin, shared tools and all other iterations. No engine code changed.

## References and interpretation

Handoff 104's shared charge-attack images and index entry were still absent, but `MTW2_Mace_charge_attack` was available in the existing local pack. Rendered both dismounted feudal knights and swordsmen militia inside this iteration using a local in-memory index adapter. Inspected both full sheets and all 33 ordered GIF frames per unit. No shared tool files changed. All raw reference material, images and numeric results remain ignored under `local_reference/`.

The reference shows a sweeping cut, while the handoff describes the engine move as a charging stab. Sent an optional preference question; with no answer, stated the default of following the reference sweep. This candidate remains subject to visual approval. Before measuring, recorded four correspondences per unit: start 0 s/frame 0, windup .2 s/frame 9, contact .4 s/frame 15 and end 1 s/frame 32.

The original reference marks impact at .75 s before the blade sweeps forward. Our contact must land at fighting reach at .4 s, so the contact pose deliberately differs. The reference also travels 4.355759 m over 1.6 s; that running stride and deep lunge are not transferred. Numeric differences are reported individually, not presented as an all-green match. No reference rotations or motion curves were copied.

## Implementation

`knight.charge_attack.json`: 21 samples at 20 Hz, one second, contact .4 s, no loop, zero external travel. The opening copies charge v2 exactly and the ending copies canonical ready exactly. The rig still has 19 joints and 79 floats per sample.

Pelvis and chest share pitch so the torso stays straight through the forward lean. Independent yaw provides the windup and sweep. The pelvis drop has explicit Hermite derivatives to bend the initially straight support leg gradually; this removed an initial between-frame foot dip of about 6 mm. Left foot swings .025–.30 s into its ready contact, right foot swings .55–.95 s into its ready contact. Each swing uses quintic progress and .08 m squared-sine lift. Knees point outward under load. Reducing the pelvis turn also removed the original knee-height leg crossings; remaining intersections are documented below.

Arm splines use global upper-arm rotation and parent-relative forearm rotation. The right arm uses bend-plane frames plus a .30 s intermediate control around the weapon side. Without that control the quaternion shortest path went across the body and exceeded the turn-speed limit. The blade frame is transported along its authored path, its edge aligns with the cut velocity at contact, and the same orientation is set on hand and weapon. The shield forearm selects the closest attainable board facing under the existing mount. Helpers are baked into saved samples. The build is byte reproducible.

## Measured result

- Contact tip: (-.324263, 1.160392, 1.706602) m, within the requested 1.4–1.8 m forward reach.
- Shared saved-clip validator passes; fastest joint is hand_r at 1349.681 degrees/s, below 1500.
- Dense checks: 201 runtime-interpolated poses on all levels. Ground penetration at most 2.368 mm; no airborne phase. Four-point planted support drift 3.108 mm left and 3.379 mm right. Maximum analytic swing-target discrepancy is 14.001 mm, distinct from planted support drift.
- Width peaks at 1.614753 m at .29 s; forward reach peaks at 1.723698 m. L0 rear extent -.716365 m. Rest height 1.8 m; triangles unchanged at 2768 / 678 / 250 / 52.
- Blade/arm, weapon/body, shield/body and weapon/shield crossings are zero over 201 phases on L0–L2. Peak leg/leg counts are 8 / 17 / 0; coat/leg 126 / 146 / 77. Counts do not measure penetration depth. `surface_validation.json` localizes the peaks and both endpoint baselines.
- Neighbor checks use 17 times, four directions, four spacings and ready/standing/three attack phase arrangements. Side neighbors clear at 1.4 m on L0–L2; 1.05, 1.0 and .9 m can intersect. The blade enters the rank ahead. Fixed relative offsets do not prove clearance during a moving charge.

Source hashes are unchanged. Clip SHA256: `df3ad51f72cd5a7cbdc69a7c6fe881c954ce61923fb04055cfee3ebd80b63679`. Charge v2 SHA256: `c99b746b464356b8f1be47e9903bd0047ccc4693998b9faadb11b6e4aeebb16f`. Model SHA256: `2df8514e28bdf73941bfdc8d6c354b78a154be6e8809fa3ca7a387b4c9c8acc8`.

## Review and integration status

Seven-view L0 frames and fixed-camera movies are rendered with the game's CPU skinning. Host checks found running games between render batches, so remaining Blender work waits while CPU checks and documentation proceed. Final review-media validation and actual inspected playback captures will be recorded below when complete.

Integration note: `work/notes/064-knight-charge-strike-for-claude-2026-10-11.md`. The opening contains neutral legs, so runtime blending from the current running phase still needs testing. Do not layer gait, bob or the old charged stab over this full-body attack. Match the sim contact to .4 s and return to ready at 1 s. No in-game test or asset approval is claimed.

Animation scripts stay in the private iteration, per the current AGENTS.md. The previous automatic review rejection of individual-file staging remains unresolved; no staging retry or archive commit is claimed. Unrelated work remains untouched.

## Review completion

Completed all review media: seven-view L0 sheet/GIF, six-pose seven-view sheets on L1–L3, matched contact comparison, fixed oblique and gameplay movies (800 x 600, 144 frames, 6 s), half-speed whole strike (640 x 640, 72 frames, 3 s), and the smooth two-direction orbit (640 x 640, 576 frames, 24 s). All videos are 24 fps. Full decoding passes. One render launch stopped before rendering when a new game appeared between the outer and inner process checks; retried after it exited. No game was killed or launched by this work.

The review has 27 local links plus eight reference-overlay links, all present; 23 linked images verified. Separate headless Firefox playback advanced through a full cycle and loop for all four movies. Inspected the four level sheets, the page layout, 80 timed fixed-camera preview captures, all 64 half-speed captures and all 54 orbit captures as ordered montages. The full figure stays in frame. This is browser playback with timed-frame inspection, not a claim of continuous video perception or an in-game motion test. Ready-to-charge replay cuts are explicitly labeled and are not part of the non-looping clip.

Final surface localization: L0's eight leg/leg peak crossings at .38 s occupy a small attachment patch with bounds (-.027142, .735224, .094097) to (-.026828, .737720, .096833) m. Both L0 endpoints have zero leg/leg crossings. L1 peaks at 17 against 12 already in ready; L2 stays clear. Coat crossings at charge / peak / ready are L0 68 / 126 / 73, L1 104 / 146 / 122, L2 72 / 77 / 64. Counts are not depth estimates.

Added the `charge_attack_v1` iteration row as awaiting review. Final review: `assets_dev/knight/charge_attack_v1/review.html`. The clip and all accepted source hashes remain unchanged after rendering and formatting. Implementation and offline review are complete; visual approval, archive staging permission and runtime integration remain separate.

## Charge strike v2: remove the second windup

Gota's review of `charge_attack_review0.png` and `charge_attack_review1.png` rejected frames 2–6 as implausible and asked to remove the anticipation starting at frame 1. The sword is already wound up during the charge. Frame 8 and the recovery were acceptable to retain. Created `assets_dev/knight/charge_attack_v2/` for this correction; v1 remains intact.

The original opening rotated the chest backward toward -100 degrees and carried the sword arm behind the head. Mesh-clearance and turn-speed checks did not catch that visual problem. Removed those opening targets entirely. New body and arm curves connect the exact approved charge directly to contact using global orientation Hermite curves. Root and legs are recalculated for the new body path while preserving the planting schedule. Chest facing progresses from about -32 to +40.6 degrees with no backward turn; the pelvis/chest up-axis difference is at most .901 degrees.

A first direct-orientation pass still raised the sword tip by about 8 cm. Separating blade direction from axial roll removed that rise: normalized direction interpolation carries the blade toward contact, shortest-axis transport carries its frame, and a quintic roll reaches the existing contact orientation. Eighty-one runtime-interpolated opening phases show zero upward tip steps. The sword hand rises slightly as the arm extends, but the blade tip moves down and forward throughout. The measured opening tip path is 2.016 m.

This replaces poses, not timeline slots: frames 1–7 are rebuilt within the same .4-second contact timing. The opening pose and all 13 samples from frame 8 through frame 20 are JSON-exact copies of the approved charge and v1 tail respectively. Duration remains 1 second, 21 samples at 20 Hz, zero travel and no loop. The generated JSON needs no dependency on v1 at runtime; rebuilding uses the pinned local v1 file.

### Revised measurements

The shared validator passes. Maximum saved joint speed is 1273.929 degrees/s. Across 201 runtime-interpolated phases per level, equipment crossings remain zero on L0–L2, maximum ground penetration is 2.064 mm and no phase is airborne. Four-point planted support drift is 3.108 mm left and 2.520 mm right. Maximum analytic ankle-target discrepancy is 10.751 mm. L0 width drops from 1.615 m to 1.336 m. Contact tip remains (-.324263, 1.160392, 1.706602) m, and maximum forward extent remains 1.723698 m.

Leg/leg peak counts are 8 / 17 / 0 and coat/leg peaks 121 / 142 / 77 on L0–L2. The remaining L0 leg crossings occupy a small upper attachment patch at .35 s, y=.7503–.7515 m. These edge counts are not penetration depths. Side neighbors remain clear at 1.4 m in tested arrangements; tighter spacings can intersect. The unchanged mesh, coat constraints and untested runtime gait transition remain documented limits.

Rebuild is byte exact. Clip SHA256: `696af3bb4c14234e9c9b5e27cb1d1d10efc9cb5660256e7838284332f9212070`. The accepted model, charge, ready, shared helpers and v1 clip hashes are unchanged. No engine source or public asset was edited. Note 065 supersedes note 064's opening and candidate path. The existing archive-staging rejection remains unresolved; no retry or archive commit is claimed.

### V2 review completion

All revised media are ready in `charge_attack_v2/review.html`: matched before/after opening sheet, five-view GIF, four level sheets, oblique/gameplay movies at play speed, half-speed playback and a two-direction orbit. All 22 local links resolve and 11 linked image references verify. Full decoding passes for all four 24 fps movies. Separate headless Firefox playback advances through complete cycles and loops: 64 observations per fixed-camera/half-speed movie and 54 for the orbit. Inspected ordered montages of 64 half-speed captures, 64 oblique captures, 55 gameplay captures and 54 orbit captures, plus the opening and level comparisons. The blade and figure stay in view and the changed opening proceeds directly into the retained contact. This is timed-frame playback inspection, not an in-game check.

Only the v2 private folder, knight iteration rows, this reserved devlog and note 065 changed in this revision. The graphics checkout has no tracked diff. The assets-dev repo remains on main, with the iteration awaiting Gota's review and no archive staging attempt. Devlogs backed up to `/data/ggando/flanks/devlogs/`.

## Charge strike v3: restore windup and repair the sword arm

Gota rejected v2's direct opening as unnatural, then asked to return to v1, keep a windup and fix the implausible arm using the M2TW reference. Created `assets_dev/knight/charge_attack_v3/`. Earlier iterations remain intact.

Inspected both full reference sheets and all 33 ordered right-side/top frames for dismounted feudal knights and swordsmen militia. The shared source clip is `MTW2_Mace_charge_attack`. The reference extends the elbow toward the sword side as the chest turns. V1's global upper-arm spline and independently timed chest motion instead pushed the humerus far behind the torso in frames 2–3. Its maximum rearward azimuth relative to the chest was 59.609 degrees. Passing mesh-intersection and angular-speed checks did not establish that this articulation looked plausible.

The revision authors the upper arm relative to the chest and forearm relative to the upper arm. The .20 s windup control uses a common bend plane and approximate directions from one reference pose, expressed relative to the chest. Rotation splines connect charge, windup, contact and the following sample. This keeps the elbow beside the torso as it turns. The original blade orientation is retained in model space; the local hand rotation is recalculated around the repaired arm. There is no transferred reference curve.

Only four sword-arm channels in samples 1–7 change. Sample 0, all non-sword-arm channels and samples 8–20 are JSON exact against v1. This restores the whole v1 body, footwork, shield, contact and recovery. The old .40 s contact and 1.00 s duration remain. Build SHA256: `4df7b12f50534deb7b4d6eeb11a029261cbc25f0205c6e21315a913a6077f133`.

### Reference measurements and limits

Kept the same four event correspondences for both units: our 0 / .20 / .40 / 1.00 s against reference frames 0 / 9 / 15 / 32. At windup, chest-relative direction errors are .064 degrees for upper arm and .151 degrees for forearm. Elbow bend is about 40.1 degrees against 40.2 in the reference. The maximum rearward shoulder azimuth across 81 opening samples is 27.797 degrees. This metric is not a clinical joint limit.

The full model-space pose still differs: upper-arm error 34.3 degrees, forearm 44.8, chest facing 24.6 and chest pitch 37.0 degrees. The reference leans backward and turns further; this revision retains v1's body. Every OVER row is exposed with an explanation in the ignored local reference review. The score is not reported as an all-green match. This targeted correction follows the reference's arm/body relationship while keeping the requested base animation.

Shared validator and deterministic rebuild pass. Maximum saved joint speed is 1273.929 degrees/s. Dense checks cover 201 phases per level, with zero equipment crossings on L0–L2. Ground penetration remains at most 2.368 mm, no airborne phase, support drift 3.108 / 3.379 mm left/right. Contact tip remains (-.324263, 1.160392, 1.706602) m. Peak width grows from v1's 1.615 m to 1.655 m; forward extent remains 1.724 m. Side neighbors clear at 1.4 m in all tested arrangements, while tighter spacings can intersect. Existing coat/thigh crossings and the untested gait transition remain documented limitations.

The play-speed render and L0 sheets are complete. Inspected the full opening from four views and 32 consecutive rendered frames from each fixed camera, covering the strike and return. A running game appeared after the world render, so remaining Blender work waits. No game was killed. Final review playback results will be appended after the media finish. No public asset, engine source, shared helper, accepted pose or model changed. The prior staging rejection is still unresolved; no staging retry or commit is claimed.

V3 play-speed preview is available at `charge_attack_v3/preview.html`, with both M2TW GIFs and the full measured comparison page. Separate headless Firefox playback verified both movies through complete cycles and loops; inspected all 40 timed captures from each camera. The repaired windup and retained follow-through stay in frame. Remaining renders still wait for game PID 1756839. This records only completed play-speed review, not completion of the pending half-speed/orbit/LOD media.

## Charge strike v4: wrist and elbow articulation

Gota marked the sword wrist and shield elbow in `refs/knight/charge_attack_review2.png`. V3 improved the humerus path but retained two incompatible equipment constraints: the sword's independent global frame forced the wrist to fold and rotate out and back, while the shield-facing target twisted its forearm off the elbow bend axis. The prior check measured arm directions, not those local joint frames.

Created `assets_dev/knight/charge_attack_v4/`. Both elbows are reconstructed from the arm directions in a shared bend plane, preserving authored joint positions while putting flex on the rest hinge axis. This correction blends out of the exact charge over .10 s. The sword wrist uses local controls at 0 / .12 / .20 / .40 / .45 s, with a windup target of 35 degrees swing and 155 degrees axial grip orientation. The new axial orientation stays near the endpoint grips instead of forcing a half-turn out and back. The blade follows the resulting hand frame. The shield follows the corrected forearm, so its board facing changes through the strike and recovery; trying to preserve that facing recreated the elbow distortion.

Body, root, head, legs and helpers remain JSON exact v1 values. The sword arm from frame 8 onward remains exact v1, including contact tip and recovery. Charge and ready endpoints are exact. Shield-frame correction stays active through .85 s and blends into ready by 1 s. This necessarily supersedes v3's promise to retain the shield orientation. No model, skin, accepted pose or shared tool changed.

### Joint measurements

At .20 s, sword wrist swing falls from 78.984 to 35 degrees. Across .10–.40 s, its peak falls from 111.767 to 41.539 degrees. Shield elbow hinge-axis deviation at windup falls from 112.576 degrees to numerical zero. Both elbows stay within .000003 degrees of their rest hinge axes over the main windup under runtime interpolation. The new check explicitly measures the local joints and retained channels rather than only limb directions.

The unchanged approved charge endpoint still measures 104.269 degrees of wrist swing in this rig convention. This is exposed in the review; no claim that the inherited endpoint is anatomically correct is made. The correction reaches 71.1 degrees at .05 s and 41.5 at .10 s. A change to the held charge itself remains a separate pose revision. These measurements describe the rig, not clinical joint limits.

The first wrist correction applied a bound independently at each sample and exceeded the 1500-degree/s speed threshold. Replacing that with coordinated local controls resolved the speed failure. A later correction removed the unnecessary axial half-turn too. Final shared validation passes at 1273.929 degrees/s maximum. Rebuild and source checks pass.

### Motion, references and clearance

The four M2TW event correspondences remain unchanged for both units. Windup upper-arm/forearm directions relative to the chest stay within .16 degrees. Blade direction and shield facing now have explicit exceptions for the joint correction. Reference images, measurements and overlays remain ignored locally. No reference motion curves were transferred.

Dense checks cover 201 phases on all four detail levels. Equipment crossings remain zero on L0–L2. Ground penetration is at most 2.368 mm; no phase is airborne; support drift stays 3.108 / 3.379 mm left/right. Existing upper-thigh and coat crossings are unchanged. Contact tip remains (-.324263, 1.160392, 1.706602) m.

The corrected equipment path is wider: maximum L0 width is 2.126 m against v3's 1.655 m. Side neighbors still clear at 1.4 m in all tested arrangements, while 1.05 / 1.0 / .9 m spacings can overlap. These are fixed-offset samples, not moving-crowd clearance proof. Runtime gait entry remains untested.

Clip SHA256: `7365c3a41d5908a62400e26cee3c8903fae9c866d62faa3b49dbac1e252921f8`. All review renders are complete. The new page includes matching v3/v4 close-ups of the marked joints, full-speed and half-speed playback, both M2TW examples, all levels and the orbit. Final browser and media checks follow. No archive staging retry or commit is claimed; the earlier automatic staging rejection remains unresolved.

### V4 review completion

All media and checks are complete. Review has 29 local links plus 16 reference-overlay links, with no missing targets; 33 linked image references verify. Full decoding passes for all four movies. Separate headless Firefox playback advanced through complete cycles and loops, recording 64 observations each for half-speed, oblique and gameplay, plus 54 for the orbit. Inspected every ordered capture montage, the matching joint close-ups, the opening sheet and all four detail levels. The whole figure stays in the full-motion views; the labeled close-ups intentionally show the upper body. This is timed-frame playback inspection, not an in-game test.

Final review: `assets_dev/knight/charge_attack_v4/review.html`. V4 remains awaiting Gota's visual approval. The graphics checkout has no tracked diff. Devlogs are backed up to `/data/ggando/flanks/devlogs/`.

## Provisional approval and delivery, 2026-10-11

Gota accepted charge strike v4 for now and requested documentation, commits and a handoff to Claude. The remaining visual issue is that the body appears to face roughly 90 degrees right during windup. This is a visual report, not a measured yaw value. Gota intends to revisit it later. Preserve the approved v4 bytes for initial integration; do not describe the torso as corrected or the clip as final polish.

The new approval does not change the clip. Its SHA256 remains `7365c3a41d5908a62400e26cee3c8903fae9c866d62faa3b49dbac1e252921f8`. Charge v2 stays the accepted held pose, and stab v1 is already integrated byte-for-byte as `assets/units/knight.stab.json`. No full charge run cycle or overhead was authored. Scripts stay private; only the two charge JSON clips need copying during integration.

The private iteration table records provisional acceptance. Handoff 108 consolidates all three animations, corrects the obsolete run-cycle and script-promotion guidance in handoff 105, and records gait blending, contact timing, remaining joint/cloth limits and the windup torso issue. Existing dense checks and media inspection remain applicable because no motion or model data changed. No new Blender, game or build run is needed for documentation and archival commits.

## Archive commit status

The explicit commit request on 2026-10-11 was followed by a scoped attempt to stage only enumerated files in `knight/charge_v1/`. Automatic approval review rejected it before execution because it reads AGENTS.md's line "DO NOT do `git add -A` and git add individual files" as prohibiting individual-file staging. No workaround or second attempt was made. The index is empty and no new commits were created.

Pending private archives are `knight/charge_v1/`, `knight/charge_v2/`, and `knight/charge_attack_v1/` through `knight/charge_attack_v4/`, with the updated iteration table in `assets_dev/README.md`. Stab already has private commits `c8198b19379a423c545f3d7febbca48d55657ec2` and `e6fe30f67857fe0d8923626056e7e77cbabfebe2`. Existing LFS patterns cover all review PNG/GIF/MP4 files. No unignored nonmedia file exceeds 1 MB. Raw references and frame sequences are ignored. Leave unrelated terrain and vegetation changes untouched.

Claude can integrate the accepted JSON paths and hashes in handoff 108 independently of the pending private archive commits. No public source or asset was changed in this delivery.
