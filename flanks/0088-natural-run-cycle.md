# 0088: a more natural run cycle (2026-09-23)

Branch `feat/knight-model`, commit ccec1ea. The cadence from 0087 is unchanged. This reshapes the pose inside each cycle.

## What was awkward

An offline viewer (tmp/scripts/gait_view.py) poses the knight GLB with a Python port of the shader and renders one stride from the side and three quarters, plus the sole paths in the ground frame. tmp/scripts/gait_curves.py plots hip, knee and foot angles. Both run from pose modules: gait_pose_current.py is the committed shader, gait_pose_new.py is the new one.

- The sword arm pitched ±27° at a run. With the blade held forward from the hand, the tip swept from level to straight down every stride.
- The rear leg locked fully straight for about a tenth of every cycle, knee at 0°, thigh held at -33°.
- After toe-off the foot skimmed forward along the ground for about 0.3 m before it lifted.
- With no ankle, the boot tilted with the shin and the toe dug about 7 cm into the ground at push-off.

## What changed (unit_instancing.wgsl only)

- The leg is hip, knee and ankle. Knee at 50% of the leg, ankle at 90%, both read off the knight's leg profile. The ankle bends over its own band.
- In stance the ball of the foot is planted and sweeps back with the ground. The foot stays flat until a flat foot would straighten the knee past about 20°, then the heel rises onto the ball just far enough (a closed-form solve). A runner also points the foot to push off.
- The stance sweeps from 0.6 of its length ahead of the hip to the rest behind, so the thigh stops at -18° at push-off, not -33°.
- The swing ankle follows a Hermite curve. It leaves still moving back at 30% of the stance speed and lands sweeping back at 20%. It lifts early, peaking at 40% of the swing, then reaches forward. In the air the foot is set against the shin: pointed at toe-off, square to the shin mid-swing, flat at touchdown.
- Swing endpoints are evaluated at the hip height of their own moment, not the current one, which removed a knee jump.
- Walk hip drop is sized for the ankle chain, not a straight leg. At 0.8 m/s the knee no longer lands bent 48°.
- The sword arm swings 0.18 rad at a walk and 0.12 at a run. The shield arm swings 0.06.
- The shoulders turn up to 0.10 rad against the legs, and the upper body shifts about 1 cm over the planted foot. Both ease in up the torso so the hips stay square.

## Numbers, knight at 6 m/s

| | Old | New | Typical running |
|---|---|---|---|
| Knee at touchdown | 33° | 18° | 15 to 20° |
| Knee mid-stance | 51° | 53° | 40 to 50° |
| Knee at push-off | 0°, locked for 10% of the cycle | 25° | about 20° |
| Knee peak in swing | 108° | 116° | 100 to 130° |
| Thigh at push-off | -33° | -18° | -15 to -20° |
| Thigh peak forward | 54° | 59° | 50 to 60° |

At 0.5 m/s it is a short walking step, at 1.5 a jog, from 3 a run. Code-built kinds run through the same bands. Their box legs just bend a little.

## Cost

200k knights, 40 m locked view, about 1,300 on L0. Unit pass × core clock: about 1,170 ms·MHz committed shader, 1,190 new, within run noise. Loadavg was 7 to 9 during both runs.

## Feel pass

Gota: better than before, still looks weird, probably because the upper body barely moves. Committed as ccec1ea.

## Next, on its own branch

`feat/knight-model` stays on replacing the code-built units with the Astra models (textured knight, then the other kinds). Upper body motion waits for a separate branch:

- Torso pitch with the step: a small forward nod at each landing and a rise at push-off, plus a slight roll toward the planted leg.
- A larger shoulder turn, about 10 to 12 degrees against the legs, and the pelvis turning the other way with the legs.
- An elbow on the arms, bent over a band like the knee, so the arm pumps from the elbow while the blade tip stays steady. The shield turns with the shoulders.
- The head steadier than the torso, so the helmet does not nod with every step.

Tune them in the offline viewer from the front and three-quarter views before touching the shader.
