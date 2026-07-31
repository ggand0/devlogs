# 0021 — Crowd twitch + combat facing fixed (2026-07-10)

First commit on `feat/battle-feel` (`90c0892`). Owner report: "each
soldier literally twitches every frame and doesn't face the enemy."

## Twitch: three stacked causes

1. **Dueling overlap solvers.** Inside 0.9 m BOTH a hard force boost
   (HARD_BOOST on the separation push) and the positional correction
   fired on the same overlap. Packed pairs oscillated at the accel cap —
   ~3 cm/tick ping-pong at 15 Hz, rendered smoothly by the interpolation
   but still reading as constant twitching. Fix: overlap is positional-
   correction only; the force term stays for the 0.9–1.4 m standoff.
2. **No damping in the press.** Spring energy bounced between wedged
   neighbors forever. Fix: `jam` factor (from the existing crowd value)
   scales shove authority down 60% and applies 25%/tick velocity damping
   when fully packed.
3. **Render-side stepping.** Yaw updated once per tick with NO
   interpolation (position always had prev→now lerp) → facing snapped at
   render rates; and the walk cycle flickered on/off from crowd-jitter
   velocities. Fix: `yaw_prev` column + wrap-aware lerp in the sync;
   move-amount goes through a deadband (0.12) + smoothstep.

Evidence (new `move avg` metric in the audit + overlay, m/tick):
idle 0.001, march 0.23, engaged line 0.03–0.11. And nn_min in melee rose
0.35 → 0.50: the oscillation had been driving bodies INTO overlap.

## Facing

Priority: locked wind-up target > nearest enemy in reach (from the same
scan, free) > movement direction. Fighters keep eyes on the enemy while
being shoved; routing/unengaged units face where they're going.

Plus **combat closing**: a unit with an enemy already inside reach adds a
bounded pull to ~1.2 m so front lines stay JOINED — separation's 1.4 m
equilibrium sits uncomfortably close to the 1.8–2.0 m reach edge,
especially with the new damping. This is combat execution (like the
wind-up foot plant), not steering: it can never activate without an
enemy in swing range, so orders stay sacred.

## The false regression (worth remembering)

Mid-fix, the FL_TEST_SURROUND line-control fight "stalled" at zero kills
and I chased a solver regression for two iterations. Reality (visible
the moment I looked with the new FL_CAM_X/Z knobs): the blue control
regiment hit its morale break and ROUTED — the correct game behavior
since 0018 landed. The surround test's per-capita acceptance metric
predates morale; its line side now ends by break, not annihilation.
Lesson: when a combat metric changes shape, check the morale state
before the physics.

## Round two: quasi-static clogs (`5298d19`)

Owner: better, "but they still twitch when clogged." Remaining sources:

1. The positional correction moved bodies up to 0.2 m PER TICK — renders
   as 15 Hz sliding even with interpolation. Now under-relaxed: 30% of
   overlap, 0.1 m/tick cap, and sub-centimeter corrections dropped
   entirely (settle noise, not overlap).
2. 40% of the shove force was still active at full jam. Now ZERO — a
   fully wedged mass is quasi-static and resolves overlap positionally
   only. Press damping raised to 40%/tick.
3. My own round-one combat closing flip-flopped direction whenever the
   per-tick nearest enemy changed. Now it steers toward the LOCKED swing
   target (stable across a whole swing cycle), falling back to nearest
   only when Ready.

Result: engaged-line mean displacement 0.005 m/tick (round one: 0.03 —
0.11; pre-fix the oscillation alone was ~0.03), hits/tick unchanged,
nn_min 0.66 — bodies at clean standoff. The principle that fell out:
**at high density, position-based dynamics only; forces are for open
field.** Crossfade by the crowd/jam factor.

## Knobs

`FL_CAM_X` / `FL_CAM_Z` (with FL_CAM_DIST / FL_CAM_PITCH) — full camera
pose from env for screenshot verification anywhere on the field.
