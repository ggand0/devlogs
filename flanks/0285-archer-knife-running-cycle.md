# Archer knife running cycle

Written by GPT-6.

2026-10-23. Gota requested a running animation from the peasant archer melee `run.gif` and `run_sheet.png`. The first candidate is in `assets_dev/archer/melee_run_v1/`, with `review.html`, `archer.melee_run.json`, authoring and review scripts, numeric reports, videos and L0/L1 pose sheets. It is awaiting review, not committed or integrated into the game. The graphics branch is `feat/archer-melee-run`, created from main. No tracked game files were changed.

## Motion

- Knife-out run, 17 samples at 20 Hz, 0.80 s, 2.72 m forward travel, 3.4 m/s at model scale.
- Right support at phases 0–0.3125; left support at 0.5–0.8125. Both intervening intervals are airborne. The foot rolls onto the toe at push-off.
- The recovery foot stays behind the body before the knee comes forward. Torso lean varies from 24.5 to 27.5 degrees; the head leans 16 degrees. Compact asymmetric arm swings retain the approved dagger grip.
- The first internal pass was too upright and advanced the recovery leg too soon. Multi-view inspection led to the stronger lean, lower hands and delayed recovery path.
- Contact paths were authored independently using ankle IK, explicit velocities and leg-reach checks. The build checks 501 analytic poses against the 0.819 m leg reach. Four reference events only were compared, with reports and overlays kept in ignored `local_reference/`. No source animation curves were copied.
- Runtime quaternion interpolation initially sank a sole by about 8 mm between 20 Hz samples. A least-squares fit of root height against actual interpolated support positions reduced penetration to about 3.1 mm. This has a small residual contact error, recorded below; it is not exact foot locking.

The approved model remains byte-identical: SHA256 `f344e31139e4648cecaea55a343427fd7e70c00b902a88d24ea7a52edb56eb80`. It retains 2,932/800/240/60 triangles, 1.80 m rest height, the 22-joint rig, hood, scarf, straight quiver and knife mount. Input hashes include the approved standing pose, mount settings and shared skeleton libraries. The pre-existing modification to `_skel/author.py` and untracked `_skel/bvhsource.py` were not edited or staged.

Clip SHA256: `3def9300e7de54bc806a21187034665de19a52bbc36985126bf028774c4fb3ef`.

## Measurements and limits

`check_run.py` samples 401 interpolated times on all four levels. Endpoints match exactly; quaternion length error is below 5.7e-8 and coat helpers match the standard bake. Peak joint speed is 871.3 degrees/s at the thigh, below the generic 1,500 degrees/s limit.

| Measurement | L0 | L1 |
|---|---:|---:|
| Maximum width | 0.7112 m | 0.7081 m |
| Forward extent | 0.6770 m | 0.6773 m |
| Rear extent | 0.6783 m | 0.6739 m |
| Maximum visible ground penetration | 3.092 mm | 3.092 mm |
| Planted toe displacement from its landing position | 8.943 mm | 13.660 mm |
| Toe error relative to its fixed ground target | 5.684 mm | 10.684 mm |

L1's simplified front sole is approximately 5 mm above L0's. The full-body loop velocity discontinuity is reported separately in the JSON; matching endpoints does not make piecewise quaternion interpolation velocity-continuous.

At 33 sampled phases, the blade clears both the body and arms on L0/L1; forearms clear the torso. Existing upper-thigh overlap stays inside the waist. L1 has many more crossing edges in that small concealed region at one phase; edge counts do not measure penetration depth.

The long tunic's rear hem intersects the recovery shin at the deepest folds, around 0.40–0.51 m height in the first half-cycle. Close-up front, rear and both side renders are included as `hem_detail.jpg`. Lower recovery heels were tested down to 0.20–0.30 m; they did not eliminate the crossing, degraded the reference fold and eventually made landing targets unreachable. Those alternatives were not retained. The approved mesh and standard automatic coat helper rule were not changed. This remains an explicit cloth limitation of the candidate.

Neighbour checks use nine phases at 0.9, 1.0, 1.05 and 1.4 m spacing. Both near meshes clear lateral neighbours at the sampled same/opposite phases. Different phases in adjacent ranks can intersect at the tighter spacings. Closing on a stationary soldier intersects even from a 1.4 m initial gap. The clip supplies no movement avoidance.

The generic validator does not pass: it sees four intentional airborne samples, three landing/end samples at 5.10–5.68 mm above ground, and the hidden lower bow limb up to 59.96 mm below ground. `generic_failures_explained.json` identifies those samples and part 15. The equipment-filtered running check reports visible geometry separately. Do not call the generic result clean.

## Review evidence

The package contains fixed front, rear, left, right, oblique and game views; a 24-second full orbit in each direction (12 seconds each way); three-stride travel videos from side, oblique and game cameras; and nine-pose, seven-view L0/L1 sheets. Overhead faces down on our sheet and up in the reference; this orientation difference is stated on the page.

The first world-camera framing cropped the path. Increasing the fixed scale from 10.5 to 13.5 fits the full projected travel with a maximum screen fraction of 0.889. Sheets, fixed views and orbit also fit. The hem close-ups intentionally crop the upper body.

All five MP4s passed ffprobe/frame-count checks and full ffmpeg decode. An isolated, GPU-disabled Chrome played every video at normal speed through its loop and captured eight ordered views per video. I inspected the ordered 24-frame side sequence, front/rear/side pose sheets, both detail levels, hem close-ups, orbit captures and side/game world-travel captures. These inspections and automatic playback are recorded; they are not an in-game test.

Game restarts repeatedly interrupted render availability. Gota provided a short free window and the final Blender batch finished. One early command printed an active game but did not gate the next command; Blender exited on a missing SciPy import before scene setup. The authoring spline was replaced with a local NumPy Hermite implementation, and subsequent launches used a checked process guard before Blender plus a second guard inside the renderer. No game process was killed.

## Integration after review

The current game still uses procedural running. The candidate needs a knife-out run slot and distance-driven phase, using model-scaled 2.72 m per cycle. Hide bow/arrow parts 8, 13, 14, 15, 16 and 17; show knife part 20. Use the whole clip without additional procedural gait, lean or bob. Movement supplies world travel once.

Start/stop transitions must handle arbitrary stride phases and the support foot. The loop ends in a running pose, not the standing/ready pose. Bow-out running and L2/L3 game playback remain separate integration work. Shared engine source was not edited. After visual approval, archive the iteration in the private asset repository and ship the JSON only; authoring scripts stay in the iteration.
