# 0009 — M6: Combat (2026-07-08)

## What landed

- **Damage** (movement.rs, folded into the existing neighbor loop — zero
  extra spatial queries): melee is a *gather*: each unit counts enemies
  within `MELEE_RANGE` (1.7 m — deliberately wider than the ~1.4 m
  separation standoff so front rows fight across the gap) and takes
  `min(attackers,4) · dps · dt · rand(0.7..1.3)` damage. Gather form = no
  write races, stays inside the parallel integrate pass. `hp` column added
  to the SoA (100 per unit). DPS default 18, `FL_DPS` env override.
- **Deaths** (`src/combat.rs`): serial pass after the sim step swap-removes
  dead units from every SoA column, keeping the selection mask in sync
  (mask swap-removed identically; cleared if stale). `CombatStats` tracks
  kills/alive per team; overlay + periodic log show both.
- **Death craters**: 0.4 % of kills carve a 4.5–8.5 m crater at the corpse
  (≤2 per tick as a remesh guard).

## The emergent showpiece

A sustained front line kills thousands in a narrow band → the 0.4 % crater
sprinkle accumulates into a **trench, then a canyon, along the front**.
End-of-battle terrain shows a jagged chasm tracing where the line stood,
pockmarked flanks where salients fought. Deformation + combat + frontline
produce WWI landscapes with no code aware of any of it.

## Acceptance

- Full 100k battle (FL_DPS=120 for wall-clock): 100k → 3.6k over ~5 min,
  orange winning 1987 vs 1629 — 96.4 % casualties, winner clear.
  Peak kill rate ~1400/s at full line contact. FPS: 124–252 during the
  heaviest phase (dips into the 50s are desktop-compositor noise; sim tick
  peaked ~7 ms early and fell as buffers compacted; 405 fps / 2.5 ms
  observed once the compositor let go).
- The asymptotic tail (last ~3 %) is scenario pacing, not engine: tiny
  remnant pockets have tiny contact perimeters. The test script re-targets
  idle groups at the enemy centroid every 15 s as a player/AI stand-in;
  true finish wants rout/morale — deliberately out of scope.

## Notes / future

- Buffer compaction via swap_remove is O(1) per death and invisible at
  1400 deaths/s; instance buffer just shrinks (rebuilt per frame anyway).
- Interpolation across a swap-remove is consistent (pos/pos_prev swap
  together); the swapped-in unit renders correctly the same frame.
- Endgame ideas when in scope: morale/rout, group merge below a size
  floor, corpse decals, kill feed.
