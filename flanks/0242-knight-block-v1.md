# Knight block v1

Written by GPT-6 Astra.

2026-10-13. Handoff 115, knight block only. Gota approved starting this clip and separately approved adding defend support to `assets_dev/_m2tw_refs/pose_check.py`. The iteration is `assets_dev/knight/block_v1/`; the private assets repository stays on main. No commit or game integration before the visual review. Knight parry and both spearman clips remain separate review steps.

## Motion and sources

The shield rises by 0.20 s, holds exactly through 0.70 s and returns to the exact shipped ready sample at 0.90 s. Nineteen samples at 20 Hz, 79 values per sample, non-looping, zero travel; `events.guard = 0.2`, `events.release = 0.7`. Both soles stay planted. The chest adds three degrees of forward brace; the sword arm retains its ready global directions. The shield remains fixed to the forearm rather than receiving an independent board rotation.

Model SHA-256: `2df8514e28bdf73941bfdc8d6c354b78a154be6e8809fa3ca7a387b4c9c8acc8`. Ready SHA-256: `e78053c67b192fef7ccca0c7f9b5abcd73de4b75b2be2c23c8e85d8fd33d7f1b`. The source copies and the graphics tree's shipped copies matched before authoring. Clip SHA-256: `31c87fed3ccfc0040b32bd10d3e2a8e43e86ce157f742830a32b71cb56875de3`.

The original knight defend_mid_stab and defend_hi_stab sheets and ordered GIF frames were inspected. Matching militia imagery was missing, so the existing reference renderer produced it under the iteration's ignored local_reference directory. Its sheets exposed the same lift and planted support. Comparison events were written in correspondence.md before measuring: ready start, guard, ready end; three events per reference. Guard comparisons use mid-stab frame 4 and high-stab frame 3. No reference curves were transferred.

The shared pose checker resolved only basic and attack clips, despite the defend catalog already existing. With explicit permission, its reference lookup now includes `index(skeleton, "defend")`. The modified lookup ran the knight and militia defend comparisons, an armored-sergeant defend pose and the knight ready pose. No other shared tool changed.

## Measurements

The shared validator passes: peak turn 936.171 degrees/s at upperarm_l; exact raw saved endpoints; coat helpers within the validator tolerance. The constructor reproduces every saved clip field. Source hashes remain unchanged.

L0/L1/L2/L3 triangle counts are 2768/678/250/52. Across 201 interpolated poses on every level, sole drift is below 0.001 mm and ground penetration below 0.001 mm. Maximum L0-L2 width is 0.949324 m at sample 0, the ready pose; L3 reaches 0.846549 m. Maximum forward reach is 0.854979 m during the transition. All 19 saved widths are recorded per level in guard_validation.json.

At the guard, the L0 shield normal is approximately (0.0294, 0.1960, 0.9802): yaw 1.718 degrees left of forward, upward tilt 11.305 degrees. Shield bounds are x -0.166 to 0.307 m, y 1.031 to 1.693 m, z 0.627 to 0.802 m. Its top sits 23 mm below the helmet crown. Bidirectional vertex-to-triangle gaps at the guard are 0.374910 m shield/helmet and 0.444508 m sword/shield. These sampled vertex-to-surface distances are not exact edge-to-edge minima.

Equipment checks at 201 phases on L0-L2 find zero blade/arm, weapon/body, shield/body and weapon/shield triangle-edge crossings. Existing coat/leg counts remain constant through the clip: 73/122/64 at L0/L1/L2. L1 retains 12 upper-thigh crossings; L0 and L2 legs have zero. The raised shield sleeve has a visible elbow crease. L3 has no connecting shield-arm mesh, so its lifted shield appears detached at enlarged review scale. These rig/mesh limitations were shown rather than changing the shared model.

Formation tests use 19 phases, equal facing, both lateral neighbors and the ranks ahead/behind; neighbors stand, hold ready, block in phase or half a clip apart. L0-L2 clear at 1.0, 1.05 and 1.4 m. At 0.9 m there are lateral and forward-rank crossings. The tests sample triangle edges and do not establish continuous clearance or arbitrary-facing spacing.

## Reference differences

The pose-check thresholds were kept at 12 degrees / 5 percentage points. Every over-threshold row has a reason in local_reference/checks.html. Knight guard RMS is 40.99 degrees against the mid block (13 rows over) and 24.44 degrees against the high block (6 over); militia is 40.84 (12 over) and 25.11 (6 over). This is not an all-green reference match.

The biggest difference is the shield forearm: this model mounts the board with the forearm directed forward, whereas M2TW carries it across the body. The ready baseline already differs by 62.6 degrees. Keeping the board strapped and facing forward lifts the elbow farther than the reference; high-reference forearm error is 67.8 degrees, mid-reference 91.4 degrees. The upper-arm error is 37.7 degrees against high, 78.4 against mid. The guard intentionally covers face and chest rather than reproducing the lower mid guard. Its shield stays more vertical than the high reference's overhead defense. The ready feet, pelvis and weapon-arm directions remain fixed rather than following the reference's deeper crouch, pivot and recoil. Visual acceptance is still Gota's decision.

## Review and verification

Review: `assets_dev/knight/block_v1/review.html`, served locally at `http://127.0.0.1:8765/review.html`. Five fixed-camera views (front, both sides, oblique, gameplay), each full and half speed, plus the 24-second clockwise/counterclockwise orbit. Ready/guard pairs, seven-view motion sheet, all detail levels and both reference units are included. Videos repeat with a 0.60-second ready pause; the clip itself is non-looping.

All 11 MP4s fully decode and play through a loop in a separate headless Firefox. The main page's 28 local links and reference page's 13 links resolve; 24 linked images verify. Ordered browser captures of the front, left, game-camera movement and orbit were inspected, along with all saved raise/return frames and an uncropped orbit capture. This is browser playback plus ordered capture inspection, not observation through a continuous video feed. Both transitions remain in frame; the feet stay fixed. The model's shield-arm bend remains visible for review.

No game was launched. Game activity was checked on the host before Blender and authoring runs; two existing games caused work to pause until the next clear check. No process belonging to someone else was stopped. The graphics tree gained an unrelated untracked target-merge directory during the session; it was left untouched. No src file or shipped asset changed, so no sim hash or cargo run applies.

All scripts, reports and media stay in the new iteration. M2TW output, frame sequences and atlas output are ignored. The iteration row is added to assets_dev/README.md as awaiting review. Nothing is staged, committed or pushed. The shared pose-checker change and this iteration should be archived together only after acceptance; game delivery and the handoff to Claude follow that review.
