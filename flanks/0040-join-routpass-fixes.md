# 0040 — Join-the-fight, pass-through, and the trickle-freeze

Date: 2026-07-19. Branch: feat/m2tw-melee at 79098ad. Owner defect
reports: (1) a rear unit ordered onto an enemy already fighting an
ally never advances — it deals with the overflow trickle instead;
(2) a unit "makes stupid space" for a retreating unit and ends up
with a wrecked formation. Both fixed with engine-documented
mechanisms only; backups work/backups/cascade-full-2026-07-19.bundle +
backup/m2tw-melee-a5be67e taken BEFORE the work.

## Root causes

1. The destination freeze keyed on ANY engagement (one wind-up,
   target-agnostic): the joining unit B froze its order the moment
   overflow men poked it, then jammed dead behind ally A with no way
   around (no lateral routing exists).
2. Same-team courtesy rest (1.4 m) let every fleeing/passing man
   excavate a ~3 m corridor; on this branch my body-scale jam ALSO
   counted the settled interface enemies, chronically fading formed
   men's slot-keeping — yield plus no spring-back. Mine.

## The fixes (evidence ledger)

- Freeze gates on a per-ordered-target engagement COUNT
  (ENGAGE_LOCK_MIN 10 in-cycle men, bridged like `engaged`). Engine:
  per-enemy engagedUnits/engagedSoldiers records; engagement gated on
  "enough soldiers in the proximity zone". A trickle never halts the
  march; the poked men defend individually.
- Attack approach routes AROUND friendly formed blocks: flank
  waypoint when the straight segment crosses an ally footprint,
  recomputed per tick. Engine: isBlocked/blockedBy — units are solid
  to friendlies; paths go around.
- Broken men and Move-ordered regiments collide at body scale
  (ENEMY_SEP_RADIUS) against same-team lines. Engine:
  formationMovingThrough (deliberate pass-through state); collision
  is radius-based for everyone. Pass-through pairs contribute
  rest-scale jam only — body-scale weighting made the passer crawl
  at quarter speed and faded the line's slot-keeping (measured on
  the first ROUTPASS run).
- Jam counts COMPRESSED pairs only: settled neighbors at their rest
  distance contest nothing, so the chronic interface fade is gone;
  the body-scale weight still brakes real packs (twitch stays dead).

## Measured (new repros committed: FL_TEST_JOIN, FL_TEST_ROUTPASS)

- JOIN: B closes 80 -> 9.9 m THROUGH the overflow phase (locked=false
  below the count gate the whole approach), locks at the mass,
  fights; when orange breaks, player-ordered pursuit cuts the
  routers down (auto_order at-ease units correctly stand down
  instead — that is doctrine, discovered via the first run).
- ROUTPASS: passer crosses A in ~50 s at body scale, no corridor,
  nn min 0.56-0.68 during, A disorder peaks 5.2 m then RECOVERS
  (4.56 and falling at t=58). First build: passer stuck inside
  crawling, disorder pinned at 4.38 forever.
- Symmetric 2v2: band intact under the new jam — r0-r3 at 87-100%
  in reach, r4 53-65%, r5 fighting; settled-press move avg
  0.014-0.022 (twitch floor).
- Gate-off DIR digit-identical, fourth consecutive time.

## Open

- ROUTPASS peak disorder 5.2 m is transient but real — a full-width
  block walking through another displaces it bodily before the
  spring-back. Owner's eyes rule on whether the pass reads right.
- JOIN B's fight share is small in the repro (the test's light-vs-
  light grind collapses in 20 s); longer grinds give B more arc.
- CHARGE/ROUT/SURROUND/FORM/200k not re-run at this tip.
