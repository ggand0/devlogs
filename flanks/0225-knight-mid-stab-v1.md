# Knight mid stab v1

Written by GPT-6 Astra.

2026-10-11. Handoff 104, item 1 only. Created `assets_dev/knight/stab_v1/` on the private assets repository's existing `main`. The graphics branch, engine source, accepted poses, model and shared tools are unchanged. The charge and other attacks are separate review steps.

## Clip and authoring

`knight.stab.json`: flanks-clip v1, 19 joints, 79 values per sample, 21 samples at 20 Hz, 1.00 s, no loop, travel `[0,0,0]`. Contact is 0.40 s. Both endpoint JSON arrays equal the canonical ready sample exactly. Contact blade tip is `(0.192423, 1.247297, 1.598957)` m in +X-left, +Y-up, +Z-forward coordinates. The clip is authored at the requested game timing, 0.4 s out and 0.6 s back.

Read the complete workflow, handoff 093, asset brief and setup notes. Inspected both complete stab_mid reference sheets and all 29 ordered GIF frames from the dismounted feudal knight and swordsmen militia before authoring. The four comparison correspondences were recorded before measurement: ours 0/0.2/0.4/1.0 s against reference frames 0/8/14/28. No other reference poses were numerically sampled, and no source rotation curves were copied. Derived measurements and reference imagery remain ignored in local_reference.

The sword hand draws back, the torso turns into a level thrust, the front sole stays planted and the rear heel rotates around its toe. Ready, wind-up, contact, brief extension and withdrawal are independently authored. An intermediate arm key at 0.30 s carries the hand outside the torso. Upper arm and relative forearm rotations use continuous rotation splines; hand and sword share the blade frame. Legs are solved from contact targets before helper baking.

An initial pose kept the rear foot at ready and opened the sword elbow too far. The reference check exposed both. Correcting the foot pivot and draw-back reduced the wind-up comparison to three OVER rows. Direct interpolation of the world elbow target caused a fast turn and body crossings between valid keys. The intermediate outward hand path plus joint curves removed those crossings and brought the peak joint speed below the existing validator limit. These were construction passes within stab_v1 before handover.

## Measurements

- Shared saved-clip validator: passes. Maximum turn 1239.945 degrees/s at upperarm_r. No airborne samples. Saved-pose minimum height -0.000134 mm, numerical noise.
- All four levels: 201 runtime-interpolated phases, finite vertices/normals. Triangles 2768 / 678 / 250 / 52; rest height 1.800000 m.
- Maximum width L0/L1/L2/L3: 1.294966 / 1.305610 / 1.301292 / 1.157394 m. L0 maximum forward reach 1.599944 m. L3 has no sword.
- Front sole drift at most 3.332 mm across levels; rear toe drift 3.361 mm. Between-sample ground penetration at most 2.244 mm, within the shared 5 mm tolerance. Saved targets are grounded; quaternion interpolation at 20 Hz introduces these errors.
- Minimum leg reach margin 33.319 mm. Maximum interpolated ankle difference from the analytic target path 6.232 mm. This target-path measure is distinct from planted contact drift.
- 201 phases on L0–L2: no blade/arm, weapon/body, shield/body or weapon/shield crossings.
- L0 upper-thigh crossings peak at 10 during recovery; localized at 0.782–0.791 m height under the coat. L1 has 12 already in ready and peaks at 14. L2 legs clear. Coat/leg crossing maxima: 108 / 137 / 69, versus existing ready counts 73 / 122 / 64. Counts are triangle-edge samples, not penetration depths or continuous collision proof.
- Neighbor checks: 17 times, L0–L2, ready, standing, same phase, quarter phase and half phase; both lateral directions plus ahead/behind at 0.9, 1.0, 1.05 and 1.4 m. Lateral neighbors clear at 1.4 m in every sampled arrangement; tighter spacings can intersect. The blade enters the space ahead even at 1.4 m, as expected for an attack. A soldier behind can hit this soldier when also attacking.

## Reference differences and rig limits

Wind-up RMS: 13.36 degrees against knight, 15.26 against militia, with 3 of 18 rows OVER. Contact RMS: 15.98 / 17.66 degrees, 8 of 18 OVER. Exact ready endpoints retain 11 OVER rows each, RMS 21.04 / 21.39 degrees. Every OVER row has a specific exception alongside its measurement in local_reference/checks.html. These are not all-green reference matches.

The largest remaining difference is shield facing. This model's fixed board-to-forearm relationship cannot reproduce the reference board normal while also matching the forearm direction; the axial turn picks the nearest available orientation. Do not hide this by changing the shared skin or weapon mounting. The front foot retains ready's facing; the rear turns 65 degrees to -15 instead of fully forward, keeping the toe fixed and the legs reachable. The differing foot contacts and shorter arm account for the remaining contact rows. The shield sleeve still creases where its missing elbow loop limits flexion. The coat intersections need separate geometry/rig work.

## Review and archive

`review.html` includes four MP4s, seven-view sheet, five-view GIF, all detail levels, matched event images, both local reference GIFs/sheets, and local annotated measurement tables. Fixed-camera videos are 800x600, 144 frames / 6 s each. The orbit is 640x640, 576 frames / 24 s, 12 seconds in each direction. Half-speed full attack is 640x640, 72 frames / 3 s. All are 24 fps. Framing is fixed from the full motion bounds.

All 22 page links and 8 reference links resolved; 20 linked images verified. ffmpeg decoded all four MP4s. A separate headless Firefox played every video beyond its loop while recording advancing timestamps/frame counters. Browser frames advanced by 245 / 260 / 257 / 726 for half speed / oblique / gameplay / orbit. Inspected ordered timed browser captures for the thrust, withdrawal, front support, rear heel pivot and orbit, plus the full page and an uncropped orbit capture. This is playback with ordered capture inspection, not a claim of watching a continuous video feed through a video-capable tool. No synthetic desktop input was sent.

The model hash is 2df8514e28bdf73941bfdc8d6c354b78a154be6e8809fa3ca7a387b4c9c8acc8. Ready hash is e78053c67b192fef7ccca0c7f9b5abcd73de4b75b2be2c23c8e85d8fd33d7f1b. Clip hash is d4aa62e43dda56e3ba0083d5d57f4d5b0482a36c53a8e48038ea86947cc64f06. Rebuild is byte-exact. `checksums.json` records the final package files. Review media use the existing Git LFS patterns; references, frame sequences, logs and atlas stay uncommitted.

Review status: candidate presented for Gota's review. No engine integration or in-game test is claimed. Note 061 gives the clip contract and surface constraints for Claude. Archive commit recorded below after creation.

Archive: private assets commit `c8198b19379a423c545f3d7febbca48d55657ec2` (Add the knight mid stab v1), 34 intended files including the shared README row. Seven review binaries verified as Git LFS pointers and listed by git lfs ls-files. No reference data staged. No push.
