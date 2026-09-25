# 0119 — Rear-rank creep: diagnosis

Written by Claude Opus 5.5. Date: 2026-09-25. Main at cc7b08b, diagnostic uncommitted (`tmp/backups/rear-diag-2026-09-25.patch`).

Gota's report (resources/unit_movements_debug0/1_092426.png): soldiers in the rear ranks of a fighting regiment keep making tiny adjustment moves with a small walk cycle, in loops and S-curves.

## Finding 1: FL_RECTFIGHT is off in normal play

`formation::rectfight()` is `env::var("FL_RECTFIGHT").is_ok()` and has been since it was written. Gota launches with plain `cargo run --profile opt-dev`, so the M2TW melee model (destination freeze, engaged close-ranks, pass-through, pursuit range) is not running in his battles. The project overview (devlog 0000) says "default on", which is wrong.

## How the rear ranks move today (FL_RECTFIGHT off)

Every fighting regiment carries an Attack order: the AI gives one, and `ai::auto_engage` gives one to any idle regiment within 40 m of an enemy. For an Attack order the regiment's reference point is the target's live centroid (`Groups::goal`), so every soldier's slot (`centroid + home`) sits inside the enemy block. Each tick a soldier with no enemy in reach gets:

- slot pull toward that slot, at `speed * dist / 35` (the arrive law), so a few meters off is a slow creep;
- the wide-acquire surge: every 8 ticks he scans 4 m, memorizes the nearest enemy, and adds a 35%-speed drive toward him, which for ranks 2 and 3 points through his own front rank;
- the jam brake, which only fades the drive at real compressed-body density (crowd 1.2 to 2.5); one friend's back is not enough;
- separation from the friend ahead (force under 1.4 m, positional under 0.9 m).

Desire forward and separation back balance at a slight compression with near-zero, never-zero velocity. Every time the man ahead shifts (swings, is shoved, staggers, dies) the balance moves and the rear man slides, including sideways around the friend's body. The render walks the legs for any smoothed speed over 0.06 m/s with stride proportional to speed (`walk_gate`, `gait_at`), so the creep shows as tiny slow steps.

"Only the front ranks fight" is not a rule anywhere: who fights is who has an enemy inside weapon reach (1.6 to 2.4 m). With enemy rest distance at 1.4 m (devlog 0042) that is ranks 0 and 1 in practice.

## Gota's question: what is a "slot" here?

Gota asked: if an attack order places the attacking unit's center near the enemy unit's center of mass, shouldn't every soldier's destination overlap with some enemy? He remembered an odd center-of-mass logic from when the attack order was first built.

Yes, that is exactly what happens, and it is deliberate in the current logic.

- A slot is a soldier's fixed place in his regiment's grid, stored as an offset (`home`) from one reference point for the whole regiment. His destination is reference point + offset.
- Move order: the reference point is the ordered point, so the grid lands where the player clicked. No order: it is the anchor where the regiment holds. Attack order: it is the target regiment's center of mass (the mean position of its living soldiers), recomputed every tick.
- So an attack order lays the attacker's whole grid on top of the enemy's grid. Side view, both blocks six ranks deep, attacker moving right:

```
where they stand:   r5 r4 r3 r2 r1 r0 | e0 e1 e2 e3 e4 e5
their destinations:                   | r5 r4 r3 r2 r1 r0
```

- The attacker's front rank is aimed at the enemy's back rank, his rear rank at the enemy's front rank, about 7 m ahead with his own five ranks in between. Nobody is meant to arrive: the destination only says "walk into the enemy" and bodies stop the men (the kernel comment: "The 'front line' is where that collision is").
- Intuition: the rear man keeps walking toward a spot he can never reach, and the first thing in his way is his own friends' backs, so he leans on them. Every enemy death or shove moves the center of mass, the whole set of destinations shifts, and he slides again.

On pacing: Gota recalled trying to make battles last longer at some point and wondered whether FL_RECTFIGHT being off is an artifact of that. The record does not show it. The gate has been opt-in since it was written (2026-07-18), the melee branch merged with it gated, and devlog 0042 lists everything as "gated FL_RECTFIGHT=1". The pacing change on record is the 1.5x HP bump (reverted in fde230d), and the enemy rest distance went back to 1.4 m for weapon clipping, not pacing. A decision made in an unrecorded conversation would not show here.

## What FL_RECTFIGHT=1 changes

Per-soldier physics is the same in both modes. The differences are at the regiment and order level:

1. Freeze: once at least 3% of the regiment (4 men minimum) is in a swing cycle against soldiers of its ordered target within 4 m, the anchor snaps once to the target's center of mass and the order stops following the live center. The men then hold at anchor + home with the holding pull (60% of speed, slowing over the last 10 m, 0.7 m deadzone). The slots still overlap the enemy, but they stop moving every tick. A routed target unfreezes it (pursuit, capped at 120 m from where the fight began).
2. A regiment with no attack order that gets engaged holds where it stands: anchor = its own center.
3. Close ranks every 8 s while engaged, if 6% (8 men minimum) have fallen since the last re-dress: slots are reassigned front to back, so rear men take the empty front slots.
4. Re-form on disengage: anchor = current center, the block re-dresses there.
5. Attack approaches route around friendly blocks instead of through them.
6. Routing men and Move-ordered regiments pass through friendly lines at body distance instead of shoving them apart.
7. The crowd brake counts only compressed pairs, and the 4 m lunge stays open until the crowd is truly jammed, then yields to the brake.

Rear-rank creep is worse with it on because the frozen anchor is still the enemy's center (the same overlap) and the holding pull is stronger than the attack pull (60% of speed over 10 m against full speed over 35 m).

## Gota's idea: a dedicated adjustment animation

Gota remembers M2TW having a dedicated animation for these small adjustments and suggested making one. Today every move plays the run cycle slowed down to the speed (gait.rs: "A walk mode with its own pace comes later"), so a real step or shuffle animation is a genuine gap. On its own it would not fix this case: 60 to 97% of the rear men are moving at any moment, so they would shuffle nonstop with a nicer animation. The two go together: the sim makes rear men stand and wait, and the shuffle animation covers the moves that remain (stepping into a gap, a correction of a meter or so). The animation list could not be checked against M2TW's files on this box (no descr_skeleton here).

## Second observation: two units sliding sideways while few men fight

Gota's screenshot resources/unit_movements_debug2_092426.png, taken 01:05:18 JST during the third diagnostic run (FL_RECTFIGHT off, about 96 s in): an orange spear regiment and a blue regiment stand a few meters apart across a strip of corpses, only a few soldiers fighting, and both blocks keep sliding along the line together while creeping.

Not measured yet. Hypothesis: with FL_RECTFIGHT off, a regiment's destination is its ordered target's live center, and the regiment it is in contact with need not be its ordered target (the AI spreads targets, auto-engage picks the nearest). If either one's target is further along the line, or its target's center drifts, the grid slides that way and drags the contact with it. This is the "interposed enemy" case in devlog 0041 (open item 3), which FL_RECTFIGHT's freeze does not cover either, because the freeze only fires against the ordered target.

## Gota's ruling: FL_RECTFIGHT stays off

Gota does not like FL_RECTFIGHT=1: some time after the regiments engage the blocks make weird flocking movements, and soldiers die off far too quickly. Off plays better. So the off path is the play mode, and any fix is designed and verified with it off.

## M2TW evidence: dedicated step and shuffle animations

Gota's Steam install is at `/data2/SteamLibraryFlatpak/SteamLibrary/steamapps/common/Medieval II Total War`. `data/Animations/pack.idx` lists every animation path (2402 strings, readable with `strings`, no unpacking needed). Every infantry skeleton (Mace, Spear, Pike, Halberd, 2H Swordsman, 2H Axe, Bowman, Crossbow, Musket, Knifeman) carries:

- `shuffle_forward / _backward / _left / _right`: moving in ready stance, in the soldier's own frame, so he shuffles sideways or back while still facing the fight;
- `stand_A_step_forward / _backward / _left / _right`: single steps out of the at-ease stance;
- `advance` and `retreat`: walking forward or backing up in ready stance, plus `combat_jog`, `walk`, `run`, `charge`;
- turning in place as its own animations: `ready_15_cw/ccw`, `ready_45_cw/ccw`, `ready_90_cw/ccw`, `stand_A_turn_45/90_cw/ccw`;
- `ready_idle` and morale-dependent ready idles (`ready_LF_high_morale_1..3`, `ready_LF_low_morale_1..3`), `knockback_from_*`, `knockback_move_from_*`.

So Gota's memory is right: small repositioning in M2TW is a shuffle or a step in four directions, not a slowed walk, and a soldier in ready stance does not turn to face where he steps. Ours turns to face his velocity above 0.5 m/s and otherwise plays the run cycle slowed down. The names do not say how the engine picks them (distance or speed thresholds) or how far one shuffle carries a man; that lives in `descr_skeleton.txt` inside the `packs/data_*.pack` files (the unpacker is in `tools/unpacker`, run under wine as in devlog 0056) and in the root motion of the `.cas` files inside `data/Animations/pack.dat`.

## Measured

`FL_DIAG_REAR=1` (new, uncommitted): per tick, for idle soldiers (Ready, no enemy in reach) of engaged Rect regiments, bucketed by rank from `home`, the velocity-equivalent terms (slot, surge, push / STEER_GAIN, corr / dt), actual speed, and whether a friend's body sits within 1.4 m inside a 45° cone of the direction he is driving. `FL_AUTOSTART=1` starts a normal battle from the menu without input (with `FL_DEPLOY=0`).

Run: `FL_AUTOSTART=1 FL_DEPLOY=0 FL_UNITS=4000 FL_REG_SIZE=500 FL_HEAVY_FRAC=0.4 FL_DIAG_REAR=1`, 8 v 8 regiments of 500 (Gota's battle size), AI on.

FL_RECTFIGHT off, developed fight (t = 25 to 90 s):

- every engaged regiment is under an Attack order
- ranks 2+: 60 to 97% of idle men moving at any tick, mean 0.1 to 0.4 m/s
- 97 to 100% of the movers are under 1.2 m/s: creep, never a walk
- 60 to 100% of those creepers are driving into a friend's body in the direction they want to go
- the rest (mostly ranks 0 to 2) creep on the arrive law toward a slot 1 to 3 m away

FL_RECTFIGHT on is worse: the frozen anchor is the target's centroid at lock time and the holding gain (0.6 over 10 m) pulls harder than the attack arrive law, so 50 to 100% of idle men move at 0.3 to 2 m/s. Holding regiments (no order) sit nearly still (9 to 13% moving) thanks to the 0.7 m deadzone.

## M2TW reference

Devlog 0036 [ENGINE]: the melee FSM carries `crowded`, `isSideStepping`, `readyStance`, per-friendly `collided` flags and a `blockedCounter`; "a soldier who cannot attack stands in ready stance, sidesteps, waits for room". Our soldiers never wait: they lean on the man ahead.

## Proposed next step (not built)

1. Wait for room: a soldier with no enemy in reach whose path is blocked by a friend's body stops driving and stands (still a body, still shovable). When the path clears he walks into the gap. The predicate is the physical one, a body in the way, from the separation scan he already runs.
2. Measure what creep is left (the arrive law near the mark) before deciding anything about it.
3. FL_RECTFIGHT: decided, it stays off (see Gota's ruling above).
4. The shuffle and step animations above, for the moves that remain.
