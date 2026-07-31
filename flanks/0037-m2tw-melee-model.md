# 0037 — The M2TW melee model (FL_RECTFIGHT, evidence-based rebuild)

Date: 2026-07-18. Branch: feat/rank-discipline, commit e4f0020 off the
merged main 0a3de56 (both rejected experiment commits reset away).
Design mapped 1:1 to the engine research in devlog 0036. Gate:
`FL_RECTFIGHT=1`; gate-off DIR run byte-identical to baseline.

## The four mechanisms

1. FREEZE (movement.rs orders + frontline.rs anchor snap). An ENGAGED
   attack order stops chasing the target centroid: anchor = centroid
   at the engagement edge, order goal resolves to None while the fight
   lasts. M2TW basis: path computed once + formation_hold_distance
   ("formations update after the last point"). The per-tick centroid
   chase was the blob engine — it dragged every slot grid through a
   moving fight and smeared battle lines progressively (measured
   below). Pursuit exception: a BROKEN target keeps the chase alive.
   Move orders never freeze (withdrawal from melee, DIR-test pins).
2. PRESS (spatial.rs META_PRESS + movement.rs separation). Men of a
   fighting regiment (engaged, Rect, not hold, not broken) pack
   shoulder-to-shoulder: same-team pressing pairs rest at
   WALL_SEP_RADIUS (1.05) — the proven shieldwall pair machinery on a
   new grid meta bit (bit 5; separate from META_WALL because that bit
   also arms the spear-line hazard and wall damage stats). The crowd
   yield computes from the pair rest radius, so a pressing man keeps
   his forward drive until genuinely shoulder-tight: bodies stop him,
   not the density rule. M2TW basis: soldier collision radius gates
   how many fight; surge out of the frame with guard off.
3. SLOT MEMORY: untouched — home slots always pull men with no
   reachable enemy; the frame they dress on is the anchored one.
4. REFORM (frontline.rs disengage edge + formation.rs). When the
   fight ends, anchor = centroid and a reform fires: the regiment
   dresses its square where it stands (the engine's discrete
   `reforming` action state). While engaged, close-ranks runs every
   8 s (240 ticks, same 6% threshold) with the anchor ratchet, so
   casualties feed reserves onto the fighting slots.

Guard mode = our H hold, already correct: leashed to arm's length,
keeps parade spacing (press excludes hold), maintains the frame.

## Measured

- FL_TEST_PILE (new acceptance, the owner's 6-onto-1 repro): ALL SIX
  regiments engaged continuously (5-6/6 sustained), engaged disorder
  2.4-2.5 m — a pressed fighting crowd that stays block-coherent.
  No queue, no gridlock, no merged ball. Kill pace is perimeter-
  limited (500-man victim on hold: ~33% cas at t=100) — pacing is an
  owner feel call.
- 200k (FL_UNITS=100000, FL_ENEMY_REGS=100, FL_ARMY_GAP=60), same
  timestamps: gated disorder PLATEAUS 5.9-6.1 m (t=100-148, 85-93
  regiments engaged — everyone fights); ungated baseline CLIMBS
  8.5 -> 13.7 m and keeps rising (the scattered-blob smear). Step
  7.0-7.8 ms gated (loaded box, parity).
- DIR gated: rear 2.19x per-hit (49.4/22.6), rear kills dominant,
  yaw dev 0.00. Gate-off: byte-identical to pre-change baseline
  (475/300/619, 22.2/28.7/49.6, control 103).
- CHARGE gated: wall dz -0.9 vs open -1.3, wall spears outlive open
  (434 vs 413), wall-beats-open ordering intact; both lanes grind
  slower since chargers stop pressing through after contact —
  re-baseline numbers at default-on.
- ROUT ~50% break (542->487), SURROUND line 86 (exact 0034 band),
  FORM slot err 0.68-0.75 / facing 0.00 / nn 1.06<1.37<1.80. Clippy
  clean.

## History (what died and why)

Leash (front-K-only fight) = guard mode as law — passive, rejected.
Contact-offset goal = still blob at scale (frames chased centroids).
Solid frames + slide = queue, "only the first line fights" — and the
research showed launch-M2TW shipped exactly that behavior as
anti-blob code, players hated it, CA patched it out. The model that
survived is the patched-M2TW one: surge + occupancy + slot memory +
reform, with the frame frozen at contact.

## Open

1. Owner feel pass at 100k/200k and FL_TEST_PILE. Note the TEMP 5v4
   sandbox defaults are GONE from the working tree (owner checkout +
   reset) — big runs need no env overrides now; say the word to
   restore the 5v4 TEMP defaults.
2. Pacing: all fights grind slower without press-through; if too slow
   the honest levers are physical (BASE_DMG, cooldowns), owner call.
3. Default-on: quiet-box band re-measure, CHARGE re-baseline.
