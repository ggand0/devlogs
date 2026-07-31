# 0027 — Sparse-fight stalls, rout pursuit treadmill, press slide (2026-07-12)

Commit `ff39f4d` on `feat/battle-feel`. Owner report: "two units stop
fighting around 270–300 men left, soldiers keep sliding without engaging
even though they're at the front line facing each other" + "sliding when
engaged is still not fixed."

## Repro & diagnosis (1v1 1000-man FL_TEST_FRONT, overlay log)

The zero-hit plateaus correlate EXACTLY with `BREAKS` lines across three
runs, and `move avg` during a plateau is 0.15–0.30 m/tick — those units
are RUNNING, not standing. Two distinct mechanisms, plus a latent third:

1. **Rout pursuit treadmill**: routers fled at exactly max speed;
   pursuers plant their feet to wind up (`desired *= 0.25`), so the
   runner is ~3 m gone by the strike tick — every blow whiffs
   (validated at hit time). Counts frozen 30–45 s per rout, resolved
   only by rallies or edge despawn.
2. **Sparse acquisition gap** (latent, would stall steady fights too):
   depleted formations hold their original block home-offsets, so a
   300-of-1000 regiment is a ghost grid at ~2.5 m spacing — interleaved
   ghosts leave units farther than QUERY_RADIUS (2 m) from any enemy.
   No acquisition → no closing → no combat.
3. **Press slide** (the "still not fixed" part): the walk-anim deadband
   started at 0.12 speed ratio = 0.7 m/s; press shoves at 0.2–0.7 m/s
   translated bodies with frozen legs even after the displacement-based
   fix (0026-era) — the band, not the signal, was wrong.

## Fixes

1. Routing units flee at `ROUT_FLEE_FRAC` (0.9×); pursuers skip the
   wind-up foot plant when the locked target's regiment is broken
   (cut-down at a run). Rout test: 27 killed at contact post-break, the
   remainder flees to the map edge and despawns (fled counter) — the
   test's acceptance shape.
2. `enemy_near` per regiment (any enemy regiment centroid within 60 m,
   computed in update_groups): its units, when the near scan is empty
   AND crowd < CROWD_SLOW AND on a 1-in-8-tick stagger, scan
   `WIDE_ACQUIRE_R` (4 m) and memoize the nearest enemy in `target`;
   the closing drive falls back to that memo (also crowd-gated, and
   validated as "some enemy within 5 m" since death-sweep reindexing
   makes indices unstable — any nearby enemy is a legitimate closing
   target).
3. Walk animation — took FOUR takes; the full post-mortem because the
   pattern is instructive (owner, rightly furious: "you half fix these
   bugs every time"):
   - v1 (velocity signal): missed correction-driven motion entirely —
     the integrator applies `corr` straight to position and KILLS vel.
   - v2 (displacement, ratio band 0.12–0.45): right signal, floor at
     0.7 m/s — above the entire press-shove band (0.15–0.7 m/s).
   - v3 (ratio 0.03–0.35, then absolute 0.2–3.0): ratio bands scale
     with kind max speed, so lights (9.5 m/s) kept a higher slide
     threshold than heavies; the absolute floor 0.2 still clipped the
     engaged-jostle band, and full stride at 3 m/s left slow shoves a
     10 % wiggle.
   - v4 (final, `3e05c74`): floor 0.06 m/s = 2× crowd jitter (the 0021
     metric), saturate by 1.2 m/s, and — the other half of the bug —
     the per-tick displacement is BURSTY (corr chains spike one tick,
     zero the next), strobing the walk cycle. Signal is now a ~0.25 s
     per-unit EMA (Local scratch in sync, updated pre-cull). Sync cost
     at 40k unchanged.
   The lesson: pick thresholds from MEASURED motion bands (overlay
   move-avg gives them directly), and check the signal's time-domain
   shape, not just its magnitude.

## Regression checks

- 40k front A/B (stash vs fix): nn min/avg 0.70/0.84, move avg 0.006,
  step 2.7–3.9 ms — IDENTICAL both sides. The nn avg drop vs older
  baselines (~1.16) is the morale pacing (fights last to ~67% losses →
  deeper press), not the kernel.
- Surround: pocket per-capita ~3× faster, both detachments end by
  MORALE BREAK.
- Rout: breaks well before annihilation, cut-down at contact, edge
  despawn.

## Open eye (owner playtest)

If "two steady regiments staring without fighting" recurs: hover both
and read the plaque — STEADY vs ROUTING and whether an attack marker is
up. Post-rally regiments are ORDERLESS by design (units move only under
orders); two rallied ghosts >4 m apart will stand and stare until
someone orders them — with FL_AI=0 nobody ever orders the enemy side.
That case is commander's-job, not sim bug; if it turns out to be the
annoying one, the fix is a sandbox auto-reattack toggle, not kernel.
