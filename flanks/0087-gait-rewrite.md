# 0087: gait rewrite and branch history cleanup (2026-09-23)

Branch `feat/knight-model`. Replaces the gait from devlog 0086, which never got Gota's approval.

## What Gota saw

On a move order the knights started in a slow walk that slid, switched to a run far too fast to look right, showed a reasonable run for about a second, then went back to the other two.

## Why it switched

The regiment's shared step was advanced every frame from the regiment centroid, which only updates on the 30 Hz sim tick. At 150 fps it read zero speed on four frames of five and five times the real speed on the fifth. Every man in a formed, moving regiment was pulled toward that step with a 0.25 s time constant.

- While the pull held, his legs ran at the regiment's broken rate, about 2 steps/s against the 5.6 his speed wanted. That was the slow slide.
- When his own rate outran the pull, the phase slipped a whole cycle, with the pull adding to his own rate on the way round. That was the frantic run.
- The per-soldier rate on its own, 5.6 to 6.4 steps/s at 6 to 6.9 m/s, was also faster than reads well on a 1.1 m figure.

## What replaced it

- One gait phase per soldier, advanced at `rate = 1.0 + 0.16 * speed` cycles per second. No regiment coupling, no cap. The phase is offset per soldier by the anim seed so a block does not move in lockstep.
- The vertex shader derives the rest from speed and cadence. `duty = min(full * rate / speed, 0.62)` and the planted foot sweeps `speed * duty / rate`, so a planted foot keeps exact pace with the ground at every speed. `full` is the longest stance the leg allows: hip swing 0.60 rad, touchdown at half the push-off distance ahead of the hip.
- When full stride would need a duty over 0.62 (a walk's), the stride shortens instead. Slow shoves become short walking steps, speed becomes a run, and the change is continuous.
- `run = 1 - smoothstep(0.25, 0.55, duty)` drives heel tuck, arm pump and lean. A running soldier leans as far as a charging one.
- Hip drop: a walk vaults over the planted leg, a run is lowest at mid-stance and highest in flight, both scaled by stride so standing is still.
- Legs stay two-bone with the knee blended over a band, as in 0086.
- Spears level on the charge band only again. The previous uncommitted change had levelled every moving spearman.
- Corpses carry a zero leg length and now keep their fall pose instead of getting a small knee bend.
- The march channel, `march_signal`, `RegimentGait` and `fall_in` are gone. The wall smoother's constant is now `k_wall`. GPU `Smooth` is back to 20 bytes and `RegimentRecord` is 16.

| Soldier | Speed m/s | Steps/s | Duty | Stride |
|---|---|---|---|---|
| Knight, leg 0.56 m | 0.5 | 2.2 | 0.62 | 0.61 |
| | 1.5 | 2.5 | 0.39 | 1.00 |
| | 3.0 | 3.0 | 0.23 | 1.00 |
| | 6.0 march | 3.9 | 0.15 | 1.00 |
| | 6.9 charge | 4.2 | 0.14 | 1.00 |
| Code kinds, leg 0.34 m | 8.5 to 9.5 | 4.7 to 5.0 | 0.08 | 1.00 |

The code kinds hold the ground too, but at 9.5 m/s their stance is 8 per cent of the cycle, so they bound.

## Verified

- A temporary trace of one knight through a formed march on the CPU path: cadence follows speed with no jumps, 3.8 steps/s at 5.6 m/s, easing down on arrival. The measured phase advance matched the formula every half second.
- Screens of the march and the arena charge, in tmp/shots/gait-rewrite/.
- DIR and ARCHERY fingerprints 19 of 19 equal to work/baselines. `FL_GPU_CHECK=1` on DIR: 1 of 18,300 frames off by one soldier, the frustum-edge rounding case devlog 0082 already recorded.
- Build and clippy clean on every commit of the rewritten branch.

## History rewrite

Gota asked for concise commit messages and for every comment written by the previous thread to be audited. The branch is now:

1. `6a4b730` Load unit models from glTF files (the loader, comments tightened, `owner` buffer renamed `winner`)
2. `6aae9f1` Animate a running gait paced by each soldier's speed
3. `4ba1396` Add a display scale for soldiers (FL_UNIT_SCALE, default unchanged)

Backups, made before the ref move:

- `backup/knight-model-c37ced8`: the old tip.
- `backup/knight-model-wip`: the old tip plus the previous thread's uncommitted diff, also saved as work/backups/knight-model-uncommitted-2026-09-23.patch (blob hashes checked against the original diff).
- `backup/knight-model-rewrite-worktree`: this rewrite before it was committed.
- work/backups/knight-model-2026-09-23.bundle, all refs, verified.
- `backup/knight-model-11b9b3e` and work/backups/knight-model-2026-09-23b.bundle: the tip with a fourth commit that reworded nine older comments on main that credit decisions to a person. Gota dropped it. Those comments come from commits already on public main (febfc22, fde230d, 20ba17e, e30beec, 0807bb6, 948f763), so they cannot be amended without rewriting main.

## Open

- Feel pass. Only Gota can judge motion. The cadence is two constants in gait.rs and both WGSL files (`gait_rate`), and the stride is `GAIT_SWING` and `GAIT_FRONT` in unit_instancing.wgsl.
- Slot inheritance on the death sweep. It swap-removes soldiers, and the one moved into a freed slot takes over that slot's render state: the corpse's near-zero speed and its gait phase. His legs pop to another phase and his stride dips for about half a second. The speed half is older than this branch. The fix is a stable render slot per soldier carried through the swap-remove and the GPU snapshot.
- The scale question from 0086 stands. With this gait a larger figure keeps the same cadence and takes a longer, more human stance.
