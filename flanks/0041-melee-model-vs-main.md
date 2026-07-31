# 0041 — The melee model: what changed vs main, and what's still open

Date: 2026-07-20. Branch: feat/m2tw-melee at 79098ad. This entry is
written for the project owner as reference documentation, not as an
agent handoff. Everything below is gated behind FL_RECTFIGHT=1;
gate-off has been digit-identical to main's combat stream in four
separate DIR runs.

## How melee works on main

- Every pair of soldiers — friend or enemy — rests at 1.4 m, the
  same distance as the formation pitch. An enemy block therefore has
  no gap a body can physically enter, and at most one rank per side
  can ever reach a weapon into the fight. This is the measured root
  of "only the first line fights"; it has been in the engine since
  the beginning.
- An attack order re-targets the enemy's live centroid every tick.
  In large battles this drags the whole slot grid through moving
  fights and smears blocks into blobs (200k measurement: disorder
  8.5 m climbing to 13.7 m and still rising).
- A routing or withdrawing unit shoves every friendly it passes out
  to 1.4 m — each fleeing man excavates a ~3 m corridor through a
  formed line.

## How melee works on the branch

Physical layer:
- Enemies rest at body contact, 0.95 m (M2TW: hidden soldier radius
  0.4 default, Feral-documented; bodies at ~0.8 m inside a 1.2 m
  grid). Gap-to-body ratio 1.47 vs M2TW's 1.5 — enemy soldiers fit
  through the gaps of an opposing grid, so the front 4-6 ranks of
  both sides interleave into the fighting band. Same-team spacing is
  untouched: formation spacing belongs to the slots, not the
  collision.
- Routing men and Move-ordered regiments collide with formed
  friendly lines at body scale (the engine's formationMovingThrough
  state): they slip through the seams; no corridor. The standing
  block is displaced a few meters while a full unit walks through
  and springs back after.
- The crowd brake counts only compressed pairs on one body scale:
  packed masses genuinely brake (no twitch), settled neighbors at
  their rest distance don't erode a formed man's slot-keeping, and
  a passer threading a line isn't misread as a wedged crowd.

Order layer:
- An attack path is computed ONCE, to the enemy as he stands, and
  freezes when the ordered fight becomes real — M2TW's
  formation_hold_distance 20.0 ("formations update after the last
  point", from this install's descr_pathfinding.txt). "Real" is a
  count gate: 10 of the unit's men in a swing cycle against the
  ordered target (the engine keeps per-enemy engagedSoldiers counts
  and gates engagement on "enough soldiers in the proximity zone").
  A trickle of overflow duels never halts the march; the poked men
  defend individually while the block keeps walking.
- Friendly formed blocks are solid to a marching attack (engine:
  isBlocked/blockedBy): an approach whose straight line crosses an
  ally's footprint swings around its flank and re-aims beyond.
- On disengage the regiment re-dresses where it stands (the
  engine's discrete "reforming" action state); while engaged,
  close-ranks feeds reserves onto the fighting slots as casualties
  open them.

What this looks like: fights happen at body contact across several
ranks, blocks stay legible rectangles pressed together rather than
a duel line at parade distance, envelopment happens by flow around
occupied space, and pacing is much faster because far more weapons
are genuinely in reach.

## Differences that are intuition calls, not settled facts

1. PURSUIT AFTER BREAKING THE ENEMY. Current behavior (identical on
   main and branch): a player-issued attack order keeps chasing a
   broken target — the unit runs the routers down across the map.
   At-ease AUTO-engagements already do the opposite: they stand
   down when the target breaks and hold their ground. The owner's
   recollection of M2TW is that units STAY after breaking the
   enemy. The recorded evidence is mixed: guard mode is documented
   as "no pursuit" (implying guard-off pursues), and CA's 1.2 patch
   note "units do not break formation when chasing routers" shows
   chasing exists — but how far a vanilla attack order chases by
   itself, versus cutting down what's nearby and reforming, is not
   pinned by any source in hand. If the owner's memory is the spec,
   the change is small: player attack orders complete on the
   target's break (like at-ease ones already do), and pursuit
   becomes a deliberate follow-up order. Undecided; owner's call.
2. FIGHTING WITHDRAWAL. If a locked target retreats slowly while
   keeping 10+ men in cycle, the frozen anchor stays at the old
   contact point — individual soldiers follow on their 5 m combat
   leash but the block does not. Possible fix: re-snap the anchor
   when the locked target's centroid drifts beyond some distance —
   still "path computed once", re-planned on real displacement.
3. INTERPOSED ENEMY. Fighting a regiment that is NOT the ordered
   target never freezes the path, so while a unit is bogged on an
   interposer its slot grid still tracks the ordered target's live
   centroid — if that target moves far, the frame slides through
   the brawl (the smear the freeze exists to prevent, in one
   scenario). Evidence-consistent fix: lock onto whichever enemy
   regiment the unit is genuinely fighting by count, not only the
   ordered one (M2TW's melee manager seizes any unit that is
   de-facto in melee).
4. THE COUNT GATE IS ABSOLUTE. ENGAGE_LOCK_MIN = 10 men: 1% of a
   1000-man regiment, a third of a 30-man remnant. The engine also
   tracks an engagedRatio, so max(floor, percentage) is equally
   evidence-consistent and scales better.
5. PACING. 4-6 fighting ranks kill much faster than main's single
   rank everywhere: pile fights resolve in ~a minute, charges are
   decisive, routs get run down. The honest tuning levers are
   BASE_DMG and swing cooldowns, not who fights.
6. PASS-THROUGH SHOVE. A full-width unit walking through a formed
   line displaces it ~5 m at peak before the spring-back
   (measured, FL_TEST_ROUTPASS). Whether that reads right on
   screen is a feel call.

## Verification state at this tip

Green: symmetric 2v2 band (ranks 0-3 at 87-100% in reach, ranks 4-5
fighting), JOIN repro (rear unit joins past the trickle, routes
around the ally), ROUTPASS repro (no corridor, disorder recovers),
PILE band, gate-off DIR digit-identity x4. Not re-run at this tip:
CHARGE / ROUT / SURROUND / FORM bands, 200k disorder and perf (last
measured two commits ago, before the press removal loosened
fighting crowds).
