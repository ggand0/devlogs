# Knight slash finish grip roll
Written by GPT-6 Astra.

2026-09-24. V3 was accepted as an overall upper-body motion, with a further request to turn the blade edge toward the oblique camera near frame 071. Reference: `resources/knight/knight_slash_review4.png`. After reviewing v4, Gota said the motion itself looks good. The awkward default ready/rest pose is explicitly deferred. Claude note: `work/notes/knight-diagonal-slash-v4-for-claude-2026-09-24.md`.

## Change

`tools/blender/melee/sword_slash.py` applies a 52-degree local blade-axis roll to the shared hand/sword orientation. It begins at clip time 0.33 s, reaches full roll at 0.37 s, holds through 0.41 s and eases out by 0.51 s. The profile has no runtime camera dependency. The upper arm, elbow, hand position, torso and shield motion are unchanged. The clip remains 0.65 s with a 50 ms raised hold and contact marker at 0.36 s. No engine, installed asset or whole-body changes.

The roll is an authored visual follow-through, not exact edge-to-velocity alignment: the maximum late edge/sweep angle is now 52.28 degrees. The validator reports that angle and checks the prior alignment requirement only before the finish roll (maximum 6.534 degrees). Do not describe the entire new cut as physically edge-aligned to its unchanged sweep. This is the tradeoff in retaining the accepted motion while changing its blade presentation.

## Validation

- 241 samples: grip separation 0 m, shared elbow hinge error 1.36e-15, rigid sword edge error 4.79e-16 m, exact loop closure and unchanged feet. No sword surface intersections with torso, shield or legs in sampled poses.
- Elbow flexion remains 32.29 to 114.55 degrees, continuously extending during the cut. Maximum relative grip rotation decreases from v3's 82.01 to 60.19 degrees. These are geometric checks, not proof of anatomical realism.
- Compared 521 samples against v3. Hand path differs by at most 7.22e-16 m; unchanged rotation channels differ by at most 9.99e-16. Blade longitudinal axis is unchanged at authored keys; quaternion interpolation between keys introduces at most 0.089 degrees of difference.
- The absolute face-normal/view dot product at frame 073 falls from 0.7847 to 0.0229; at 076 from 0.7723 to 0.0120. Zero is edge-on. Inspected sequential finish frames 071, 073, 076 and 079; full 108-frame previews rendered from both cameras for motion review by the user.
- Saved scene playback matches authored vertices within 1.67e-6 m. GLB is copied byte-for-byte from v3; inspector confirms part UV and RGBA attributes. L0 2836 triangles, L2 250, height 1.800 m. Pivot positions remain in `motion_validation.json`.

## Review files

- `assets_dev/knight/diagonal_slash_v4/knight_diagonal_slash.mp4`: oblique and gameplay cameras, normal speed followed by half speed.
- `assets_dev/knight/diagonal_slash_v4/review.html`: playback speed and frame controls.
- `assets_dev/knight/diagonal_slash_v4/knight.slash.json`: clip table.
- `assets_dev/knight/diagonal_slash_v4/motion_validation.json`, `v3_comparison.json`, `saved_scene_validation.json`: measurements.
- `assets_dev/knight/diagonal_slash_v4/previous_source/` and `previous_slash.json`: v3 source/table snapshot. V3 outputs remain intact.

Current source is under `tools/blender/melee/` and `tools/blender/knight/{prepare,review,check}_slash.py`. Defaults and README point to v4. Archer work stays closed; celebration deferred. Whole-body slash and other unit retargeting have not started.
