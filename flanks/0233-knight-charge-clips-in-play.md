# The knight's charge pose and charging blow in play

Written by Claude Opus 5.5. 2026-10-12, branch `feat/spearman-archer-clips` in the `flanks-gfx` tree, commit 420c8b9 on a4c29b2. Astra's handoff: `work/handoffs/108-knight-charge-animations-for-claude-2026-10-11.md`. The clips, byte for byte: `assets_dev/knight/charge_v2/knight.charge.json` (c99b746b..., one sample) and `assets_dev/knight/charge_attack_v4/knight.charge_attack.json` (7365c3a4..., 21 samples at 20 Hz, contact at 0.40 s, 1.00 s, from the charge pose to the ready pose). The strike is provisionally approved; its wind-up body facing is a known issue Gota fixes later.

## What changed

- `assets/units/knight.charge.json` and `knight.charge_attack.json`, in slots 8 and 9 (`SKEL_CLIPS`, `CLIP_CHARGE`, `CLIP_CHARGE_ATTACK`). The slot count was already 12.
- **The charge pose.** A near knight's upper body (chest, head, arms, sword, shield) holds the pose while he runs at the enemy; his own pelvis, legs and coat keep the gait, since the pose's are the standing pose's. Its weight is `confident * g.run * moving`: his regiment charging or fighting steadily, times the gait's own run measure. The arm sway stops while held.
- **The charging blow** plays its clip the way the cut and the stab play theirs: the sim's wind-up maps onto the clip up to the contact, the follow-through onto the rest. A knight's wind-up is 12 ticks plus one (0.43 s) and `FOLLOW_S` is 0.6, so the clip runs at 0.92 of its speed before the contact and at its own speed after. No code arm motion and no blow lean play over it.
- `skel_over` lays a place in a clip over the pose, the lower body by one weight and the rest by another. The cut and the stab go through it too.

## Three things the handoff did not say

1. **The charge pose cannot follow the regiment's charge flag.** The flag (the `sprint` band) ends when the regiment's first man starts a blow (`frontline.rs`, engaged ends the charge). In the charge test that is about 2 s before most knights reach the line, and the band falls to 0 within 0.35 s. On the flag, every running knight would drop the pose together and snap it back up in 87 ms when his own charging blow began. So the pose follows each man: running (the gait's `run`, 1 when both feet leave the ground) in a regiment that charges or fights steadily (`confident`, band 0.65 or 1). A wavering or broken regiment's runners (band 0.5) do not raise it, so routers do not run off with a raised sword.
2. **The legs during the strike keep the gait while he moves.** The handoff asks to suppress the gait for the whole strike. The sim plants a striker's feet only by cutting his drive to a quarter (`soldier.rs`, `SWING_WINDUP`), and his legs brake at 4 m/s² (`LEG_ACCEL`). Measured with a temporary counter in the charge test: the charging blows start at 4.43 m/s on average (the threshold is 0.6 of his run speed, 3.4 to 3.9 m/s), so through the 0.43 s wind-up he still moves at about 2.7 m/s unless he runs into a body. Clip legs at that speed would slide on the ground. So the rule is the cut's and the stab's: the lower body takes the clip by `strike * (1 - moving)`, and the clip's crouch and lunge show as he stops. Against a body he stops at once and gets the whole clip; against a router he runs through it.
3. **The clip's chest blends in the world.** Over hips that keep the gait, the clip's chest turn (relative to its own hips, which turn up to 56°) would face the upper body wrong. A joint whose parent is of the lower body now blends toward the clip's turn in the world and is set relative to the hips it has. With the whole clip (standing still) this is the clip exactly; it changes the cut and the stab only on the move and during the 0.08 s blends in and out.

## Checks

- opt-dev build, strict clippy and the 36 tests pass. The charge test runs with no error lines; the loader reports the ten clips.
- The five handoff hashes match (stab, charge, strike, model, ready). The strike's first sample is the charge pose to 0.05° and its last the ready pose to 0.04°.
- In play (`FL_TEST_CHARGE=1`, the east lane, cameras pinned by `FL_CAM_*`, recorded with `work/scripts/clip.sh`):
  - The running knights hold the charge pose; it matches Astra's `charge_views.png`.
  - With a temporary tint (green for a blow played from the clip, red for a charging blow drawn by parts), the front rank's charging blows play the clip at the near levels; 37 of about 290 first-wave blows carried the charge bit in the counter run, all within 4 s of the contact.
  - Frames of the contact: the upper body sweeps the blow, knights that stopped drop into the clip's crouch, running ones keep their stride.
- The tint and the counter were removed before the commit; `soldier.rs` and `regiments.rs` are unchanged, so the sim is untouched and no hash run is needed.
- Not measured: the pose pass cost. The added work runs only for near knights that are running at the enemy or in a clip blow; the march and fight views of the perf runs have few of either.
- Not done by me: watching the motion by feel at full speed. Review clip: `tmp/runs/charge_clip/review_charge.mp4` (untinted, the east lane from 3 s to 13 s).

## Open

- The strike's wind-up body facing (Astra's, Gota's call when).
- The L1 to L2 boundary: past L1 the knight draws the code's charge by parts, as before.
