# 0035 — Keep the rectangle in melee (FL_RECTFIGHT, owner-gated)

Date: 2026-07-17. Branch: feat/rank-discipline, commit 595aebf off the
merged main 0a3de56. Gate: `FL_RECTFIGHT=1`; default sim unchanged.

## The dead end first (owner-rejected, reverted)

The first phase-C attempt implemented the plan text literally: only
the front K ranks fought and closed; deeper ranks were leashed to
their slots. Blocks stayed rectangular but most of the regiment stood
watching the front rank fight. Owner verdict: complete BS, not M2TW.
The commit was reverted (e7f470f, dropped from the branch).

Research that settled it (totalwar.org guard-mode thread, twcenter
guard-mode threads, M2TW Heaven strategy guide, EDU modding guide):

- M2TW melee default (guard mode OFF): "soldiers push and all try to
  fight" — every man surges toward the fight, units wrap around
  engaged enemies, players turn guard off to maximize 1v1 duels. The
  formation loosens in the press and the unit RECOVERS its square
  when winning / after combat.
- Guard mode ON is the optional defensive stance: each man stays in
  his place, the first line fights, no chasing. That is what the
  rejected leash was — and it is what our H (hold) stance already
  does (fight only what steps into reach).
- Formation NEATNESS is the EDU `stat_mental` TRAINING value
  (untrained / trained / highly_trained): how neat the ranks stand
  and how far a soldier may wander out of formation. Most units are
  "trained": a nice square that loses cohesion when moving/charging.
  Discipline (impetuous/disciplined) is morale response + charging
  without orders, not neatness. If we later want per-kind strictness
  (heavies neater than lights), training is the M2TW-faithful lever.

## What actually shipped

Everyone fights exactly as before — no gating of who engages. Two
changes only, both about the BLOCK, not the men:

1. **Engaged attack orders stop at contact** (movement.rs). The order
   goal was the target CENTROID, so the attacking block's slots
   translated through the enemy block and both rectangles dissolved
   into one mush — that was the real mush source all along, not the
   men chasing. Once engaged, the goal becomes the point where my
   front rank meets the target block's near face: rectangle support
   of the target along the approach (from the `front_fwd`/`rank_pitch`
   geometry baked at slot assignment) + my front half-depth + a 1 m
   stride. The goal keeps tracking the target, so the block advances
   as the enemy front collapses and recedes. Charge approach (not yet
   engaged) keeps the centroid goal — the phase-B impact needs speed
   at contact. Move orders untouched (withdrawal from melee, the
   DIR-test pins).
2. **Engaged close-ranks** (formation.rs): Rect regiments re-dress
   every 8 s mid-fight at the existing 6%-losses threshold, snapping
   the anchor to the centroid — M2TW units recover their square
   during a melee they are winning.

Plus a `[rectfight]` log (4 s): mean disorder of engaged regiments.

## Measured (loaded box, relative reads)

- Auto arena (500v500 heavies/lane): engaged disorder 0.54–0.73 m —
  blocks keep their rectangle with everyone fighting. Front-lane
  kills 173 @ t=96 vs 155 for the rejected leash version (baseline
  mush ~350 @ t=88 — a real front-width fight is slower than two
  interpenetrated crowds; owner judges pacing). Rear lane collapses
  properly (196 left vs 244 under the leash).
- DIR: rear 2.24x per-hit (49.6 vs 22.1), control 107/500, lone yaw
  dev 0.00 — band holds.
- CHARGE: wall dz −2.0 m, trades ~1:1 with the chargers; open lane
  bowled −2.8 m, losing 2.3:1 — wall-beats-open ordering intact
  (absolute numbers shift vs 0034: engaged chargers stop pressing
  through, so fewer men feed onto the spear line; re-baseline at
  default-on).
- Gate-off invariance: DIR baseline re-run identical to 0034 bands
  before any gated work.

One startup segfault on a rapid back-to-back launch (only SystemInfo
logged, before any sim code) — graphics init flake, did not reproduce.

## Round 2: block solidity + slide (024747a)

Owner at scale: 6-onto-1 attack orders = blob (all frames aim at the
same face; nothing keeps friendly BLOCKS apart). Perimeter-division
proposal rejected as a trick — research says M2TW has no attacker
coordination either: units are physically SOLID to each other, and
envelopment is the player's job.

The law: an attack order's destination is clipped where my front
would meet the near face of ANY standing formed block in my corridor
(rectangle support, groups x groups per tick, ~0 cost). Target blocks
only once engaged (charges land at speed) and is never slid around.
First version glued blocked regiments to the queue — owner: "only the
first line fights" — fixed by the other half of collision response:
SLIDE along the blocker's face toward free ground, re-resolved per
tick. Gridlock bug found via FL_TEST_PILE: an abreast neighbor
(fractionally ahead) counted as a wall and clusters sidestepped
forever — a block is only a wall if its near face is AHEAD of my
front (s >= -2 m); merged/abreast blocks are bodies for separation.

FL_TEST_PILE (new acceptance, the owner's repro): six blues
attack-order one holding orange. Result: 3/6 engaged = perimeter
saturation for equal-width blocks (front + both flanks), second wave
pressed a stride behind, orange 500 -> 202 in 28 s of contact, routs,
is pursued and annihilated. Arena regression identical (disorder
0.60-0.64, same kill curves). 200k battle: engaged disorder 3.5-4.6 m
vs 19 m mush before the law; step 5.4-5.6 ms. DIR 2.24x and CHARGE
wall>open unchanged.

## Open

1. Owner feel pass at 100k/200k (FL_RECTFIGHT=1) and in the 5v4
   sandbox, pacing verdict.
2. Per-kind neatness via the training model (heavies tighter) if the
   uniform look isn't enough — data-only knob once wanted.
3. Default-on: full band re-measure on a quiet box, CHARGE re-baseline.
