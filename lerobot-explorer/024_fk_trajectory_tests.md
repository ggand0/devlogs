# Devlog 024 -- FK trajectory tests with physical ground-truth validation

## Context

The app computes EE trajectories via forward kinematics (k crate + URDF) but
had zero automated tests. Before posting to r/robotics, needed regression
tests to validate the FK pipeline is correct.

Prior validation (devlog 016) was manual cross-validation of k crate vs placo
on values read from existing datasets. That proved the two FK implementations
agree, but never validated against the physical robot.

## Approach

Recorded 4 fresh joint-angle poses from the physical SO101 follower arm using
a custom script in the vla-so101 project. For each pose, the robot was
commanded to the position via IK, the actual arrived joint angles were read
back (servo compliance means they differ from targets), and the EE position
was measured with a tape measure from the base center.

Two-layer test strategy:

**Layer 1 -- k crate vs placo (0.5mm tolerance):** Feed actual_joints_deg
into our forward_kinematics_deg(), assert output matches fk_actual_ee_m
(computed by placo in Python). This is a regression test -- if someone
refactors and breaks deg-to-rad, joint ordering, or index mapping, it
catches it. Tight tolerance because both are pure math on the same URDF.

**Layer 2 -- FK vs physical reality (3cm tolerance):** Assert our FK output
is close to measured_ee_m (tape measure). Only checks X (forward distance)
and Y magnitude (lateral). Z has a systematic offset because the URDF base
origin is above the table surface, and Y sign can differ by measurement
convention. 3cm tolerance accounts for tape measure accuracy (~1cm) plus
the base center estimation error.

## Poses recorded

| Name | Description | shoulder_pan | shoulder_lift | elbow_flex | wrist_flex | wrist_roll |
|------|-------------|-------------|---------------|------------|------------|------------|
| reset | Near-home, arm forward | 3.43 | 0.62 | 8.92 | 2.24 | 92.88 |
| straight-forward | Arm extended horizontally | 1.23 | 103.74 | -85.41 | -1.54 | 92.70 |
| rest | Arm folded back/down | -0.53 | -98.11 | 94.37 | 76.09 | 92.79 |
| diag-30 | Shoulder rotated ~24 deg | 22.15 | 53.19 | -41.63 | 79.52 | 92.88 |

Joint values shown are actual_joints_deg (what servos reached, not the
original recorded targets). All in degrees (DEGREES norm mode).

## FK vs measurement results

| Pose | FK X | Meas X | dX | FK |Y| | Meas |Y| | d|Y| |
|------|------|--------|-----|-------|---------|------|
| reset | 0.373 | 0.365 | 0.8cm | 0.028 | 0.020 | 0.8cm |
| straight-forward | 0.454 | 0.440 | 1.4cm | 0.017 | 0.010 | 0.7cm |
| rest | 0.137 | 0.135 | 0.2cm | 0.007 | 0.010 | 0.3cm |
| diag-30 | 0.284 | 0.280 | 0.4cm | 0.108 | 0.120 | 1.2cm |

All within 1.5cm on X and 1.2cm on |Y|. Z has a systematic ~2-3cm offset
(URDF base origin height above table) so it is not asserted tightly.

## k crate vs placo agreement

All 4 poses: max component diff < 0.5mm. Same sub-millimeter agreement
as devlog 016, now on fresh poses from the physical robot.

## Test inventory (13 tests)

Layer 1:
- fk_matches_placo_all_poses
- fk_matches_placo_reset
- fk_matches_placo_straight_forward

Layer 2:
- fk_within_physical_measurement_tolerance

Pipeline unit tests:
- pos_indices_so101_contiguous
- pos_indices_openarm_interleaved
- pos_indices_empty_input
- pos_indices_no_pos_suffix
- compute_trajectory_uses_pos_indices
- so101_dof_is_five
- so101_ee_autodetect
- episode_data_path_v21
- home_position_sanity

## Test fixtures

- `tests/fixtures/so101_follower.urdf` -- SO101 URDF from TheRobotStudio/SO-ARM100 (16KB)
- `tests/fixtures/so101_ground_truth_poses.json` -- 4 poses with joints_deg, actual_joints_deg, fk_actual_ee_m, and measured_ee_m

## Known limitations

- Only SO101 has physical ground truth. OpenArm was cross-validated against
  placo (devlog 016) but not physically measured.
- The 3 non-reset poses have actual_joints_deg from IK-commanded positions
  with servo compliance. The reset pose has the most reliable actual joints
  (re-read after settling).
- Z axis not tightly validated due to URDF base origin offset from table.
- No integration tests against real parquet files yet (Tier 3 from plan 008).

## Files changed

- `src/trajectory.rs` -- added #[cfg(test)] mod tests (240 lines)
- `tests/fixtures/so101_follower.urdf` -- new fixture
- `tests/fixtures/so101_ground_truth_poses.json` -- new fixture

## Run

```
cargo test --profile opt-dev -- trajectory::tests
```
