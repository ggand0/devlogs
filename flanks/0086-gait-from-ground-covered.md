# 0086: the legs start following the ground (2026-09-23)

SUPERSEDED by devlog 0087. This gait was rewritten and its commit replaced.

Branch `feat/knight-model`, commit c37ced8, on top of the glTF loader (devlog 0085). Gota's call after seeing the imported knight in motion: the men slide.

## What was wrong

The walk cycle ran at `globals.time * 9.0`, a fixed 1.43 Hz, whatever a soldier was doing. Legs swung a fixed 0.55 rad about the hip. A knight's leg is 0.56 m, so one step moved his feet 0.58 m and at 2.9 steps a second they covered 1.7 m/s. He travels 4 to 6 m/s. The other 60 to 70 per cent was the ground moving under a planted boot.

On cube soldiers nobody noticed. The eye had no ankle, no knee and no boot to track. On a model with all three it reads as sliding, which is what he saw.

## What was built

A soldier carries a gait phase, in cycles, one cycle being two steps. It advances with the ground he covered, so a planted foot cannot skate by construction. Both paths keep it beside the pose smoothers they already had: `Smooth` in render_units.rs, which became one struct per soldier instead of five parallel vectors, and the matching `Smooth` in unit_build.wgsl, now 24 bytes.

A formed regiment has its own shared phase, advanced by the block's own centroid speed in gait.rs, and each man pulls his own phase onto it by how formed his regiment is, with a quarter second time constant. That replaces the old trick of mixing two sine waves by a march signal. Falling into step is now the thing itself, so the shader no longer needs the march channel.

Two record channels changed meaning. `anim.y` was a 0 to 1 walk gate and is now the soldier's smoothed ground speed, with every reader deriving the gate from it. `anim2.x` was that march signal and now carries the kind's leg length, hip pivot to sole, measured at startup off the mesh that is actually drawn. That measurement is why an imported model and a code-built one both get a correct step rate: the knight's leg is 0.56 m, the code-built kinds are 0.34.

## The leg is two bones now

This is what makes it read as running rather than as fast walking.

The pose states where the foot has to be, relative to the hip. Planted, it starts ahead and sweeps back under the body. Swinging, it tucks up toward the hip and reaches ahead again. Then two bone inverse kinematics: the knee angle is `2 * acos(distance / leg)`, the thigh takes the aim plus half the flexion, and everything below the knee turns with the shin. The bend is weighted over a band around the knee, which is linear blend skinning with a procedural weight, so the mail bends instead of the leg cracking open. No new vertex attribute: the fraction down the leg is `(pivot - y) / leg`, and both numbers are already in the record.

The hip drops onto the planted leg by as much as the stride opens, because a leg is a pendulum and splitting it lowers the hip. A walk rides back up over the planted foot, a run stays low and floats between steps. The drop is capped near 13 per cent of the leg: past that a real runner extends the ankle, and this rig has no ankle, so the foot floats a centimetre at the end of a long stance instead of the man squatting.

## The number that was wrong the first time

Duty factor, the share of the cycle a foot is down. The first attempt used 0.38 at speed, which is a jog. A fast run is nearer 0.20: the foot is down a fifth of the time and the body is airborne for the rest, which is exactly why a runner covers so much more ground per step than a walker. At 0.38 the cadence came out at 3.8 steps a second with a short stride, which is a man speed-walking. Gota's word for it was nightmare fast walking, and he was right.

Cadence is whatever keeps the planted foot still, capped at about 2.9 steps a second for a 0.9 m leg and scaled by the square root of leg length, the way a pendulum's period is. Short legs step faster for the same reason a short pendulum swings faster.

## Walk versus run

Gota's call, 2026-09-23: the charge cycle looks right and is now used for ordinary movement too. The short-stride walk it used to blend to at low speed was the thing that slid, because a short stride cannot cover the ground under it. So the hip swing barely grows with speed now, about 27 degrees at a walk and 32 at a run, and ordinary movement pays for the long stride in cadence instead. A real walk cycle, with its own shorter stride and its own duty, is its own piece of work on its own branch.

## What it matches now

Foot speed against ground speed, which is what sliding is. A knight, leg 0.56 m:

| Speed | Cadence | Matched |
|---|---|---|
| 0.8 m/s, shoved in a press | 2.6 steps/s | 100% |
| 1.5 m/s, slow | 3.5 steps/s | 100% |
| 4.2 m/s, march | 3.7 steps/s | 94% |
| 6.0 m/s, charge | 3.7 steps/s | 66% |

At and below a march the feet now hold the ground. A charge is past what a 1.1 m figure can cover with a 0.56 m leg at any cadence a man turns over.

## The scale underneath all of it

Everything in this world except the soldiers is already true metric: 1.4 m between men in a formation, 1.8 m of sword reach, 120 m of arrow range, 9.81 gravity. The soldiers are 1.0 to 1.1 m. A knight therefore covers four to five of his own heights a second, which nothing his size does without a bound or a blur.

`FL_UNIT_SCALE` sets the display height, 1.0 being today and 1.64 putting a knight at a true 1.8 m. It routes through `unit_types::half_height`, the imported model follows it for free because the loader scales to that number, and the code-built meshes are scaled at build. At 1.64 a march matches 100 per cent and a charge 83, and formations read as a dense line instead of scattered figures on open dirt. It is sim-visible, so a run with it set will not match a behaviour baseline. Costs not yet measured: taller soldiers cross the level thresholds further out, so more of them draw at L0 and L1.

Light infantry at 9.5 m/s stays unmatched at any scale, 32 per cent at 1.0 and 41 at 1.64. That speed is beyond a sprint for a man of any size, so it is a sim number to look at, not an animation one.

## Gates

Direction and archery fingerprints 19 of 19 equal to the baselines at the default scale, so the sim is untouched. Build and clippy clean. The GPU and CPU paths share every curve through gait.rs, with the two of them repeated in WGSL and named in the comments as the copy to keep in sync.

## Open

- Feel pass on the run in motion.
- A real walk cycle, short stride and high duty, for ordinary movement.
- The scale decision, and its cost at 200k if it goes in.
- The swing arms still only pitch up and down, because the model's arms point forward in the rest pose. A run wants them swinging front to back, which needs either an elbow or a different rest pose from the asset track.
