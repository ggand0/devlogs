# Spearman standing v1

Written by GPT-6.

2026-10-11. Gota confirmed delivery of the spearman skeleton and shared-tool fixes, assigned standing as the first pose, and kept `_skel/`, `_m2tw_refs/` and `skeleton_v1/` outside this task. Read note 066 and the skeleton README. Recorded the intended key pose in note 053 before authoring.

## Candidate

`assets_dev/spearman/stand_v1/spearman.stand.json`, one sample in flanks-clip version 1, 20 Hz header, zero travel. Authored in `build_stand.py` on the supplied skeleton. The model and skeleton's original standing clip remain unchanged, verified by source hashes. Work was performed from the flanks-gfx root, with no main-repo code or branch changes.

The pose widens ankle spacing to 0.360000 m, turns the toes outward -9/+5 degrees, drops the pelvis 7 mm for grounded leg reach, keeps the spear upright and brings the shield arm closer to the body. The shield remains rigidly attached to the forearm, with an identity local shield rotation. The weapon hand keeps the modeled grip, with the shaft butt 16.412 mm above ground.

| Level | Triangles | Width | Forward extent |
|---|---:|---:|---:|
| L0 | 2992 | 0.878436 m | 0.318006 m |
| L1 | 664 | 0.876619 m | 0.305759 m |
| L2 | 250 | 0.876412 m | 0.304922 m |
| L3 | 56 | 0.863366 m | 0.301006 m |

Both soles remain at the ground on every level, with less than 0.001 mm error after saved-quaternion rounding. At 0.9 m lateral spacing, bounding intervals leave at least 21.564 mm between identically facing copies. This establishes clearance only for these standing poses, not different facings or transitions.

## References and attachment constraint

Inspected both full seven-view standing sheets and ordered GIF samples at frames 0,5,10,15,20,25,30,35,40. Compared only static frame 0 of armored_sergeants and peasants numerically. Reference material, overlays, tables and generated exceptions remain in ignored `local_reference/`.

The final reference comparison retains four OVER rows against armored sergeants: shield upper arm 33.1 degrees, shield forearm 38.3 degrees, shield normal 26.1 degrees, spear forearm 22.5 degrees. Peasants has the same three arm exceptions and no shield row. RMS is 15.9 and 14.9 degrees. Feet, knees, body and spear direction meet the default tolerances. The review page explains each exception rather than treating validator success as a reference match.

An earlier construction brought the forearm nearer the reference but rolled the shield sideways and increased width to 1.029 m. It was not presented or accepted. The supplied shield normal is 40.164 degrees from its forearm, whereas the reference is 88.068 degrees. Rotating the forearm cannot change that relationship. The candidate preserves the attachment and uses a more forward-facing board with a tucked elbow. Note 053 asks Claude to revisit the attachment if a closer across-body arm carry is required. The spear forearm stays higher because lowering the fixed grip would bury the shaft butt.

## Checks and visible limits

The shared validator passes. `check_stand.py` checks the saved pose using the CPU reference skin, finite positions/normals, all-level bounds and sole contacts, zero travel and fixed shield attachment. Equipment/body, weapon/upper-forearm and weapon/shield edge crossing counts are zero on L0–L2. L3 receives bounds and ground checks only.

Coat/leg crossing counts, rest to standing: L0 0 to 6, L1 44 to 44, L2 20 to 20. L0's crossings are at y=0.807–0.827 m near the upper thighs. Leg/leg counts are 13 to 13, 11 to 18, and 0 to 4, localized near the hip attachment; the point bounds are saved in `surface_locations.json`. These are measured limitations, not a declaration of zero intersections. The lower L1/L2 coat intersections remain visible model/skin limits.

A small strip beside the scabbard looked detached in orbit playback captures. Additional matching close renders of rest and standing confirmed it is already present in the supplied model. `hip_comparison.png` exposes this in the review. No shared mesh or skin was altered.

## Review and archive

`review.html` contains the original and authored pose, full seven-view sheets at 3.6 m framing, body close-ups, all levels, 5 by 5 formations at 1.4/1.05/0.9 m, both local reference GIFs and tables, and an orbit. Our five-view GIF is explicitly a static pose. The MP4 is 640 by 640, 576 frames at 24 fps, 24 seconds: clockwise then counterclockwise, with an eased reversal.

Checked host game processes before each background Blender run. No game was running at those checks. No game build or launch was needed. The video decodes completely. An isolated headless Chrome played the page through the full video loop; timed captures and playback counters establish advancement and looping. Inspected the page screenshot and captured front/side/back orbit views, plus separate stills. This was browser playback with sampled visual inspection, not a claim of continuous human viewing. Local links and unchanged input hashes pass. A sandboxed ffprobe/ffmpeg run failed with Bad system call; rerunning those read-only checks on the host passed.

The private assets commit contains only this iteration and its README row. Another task's uncommitted knight charge README row is preserved separately. Source scripts stay in the iteration until approval; no push or public promotion. Standing v1 is a review candidate. Ready and thrust have not been started.

Archived as assets_dev commit `1db4993` (Add the spearman standing pose). All seven review binaries are listed by `git lfs ls-files`. The reference folder and raw frames are ignored and absent from the commit. The graphics worktree is clean; the private repository retains the pre-existing unrelated changes.
