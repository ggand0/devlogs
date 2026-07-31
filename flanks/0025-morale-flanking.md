# 0025 — Morale: flanking dominates, allies stiffen (2026-07-12)

Commit `8888927` on `feat/battle-feel`. Owner: units routed too easily —
any clogged line tripped the "outnumbered" drain even in good order, and
holding formation should count for something. Direction: hard to know
you're outnumbered mid-battle (minor factor), being OUTFLANKED is the
real killer, and nearby allies should extend endurance.

## Steady-state drains (regiments.rs, constants at top)

1. **Casualties** (unchanged): `280 × tick_deaths/initial × resist` —
   ~35 % losses alone break a light (resist 1.0), ~60 % a heavy (0.6).
   Undamped by support — dead men are dead men.
2. **Outnumbered** (demoted): was >2:1 blurred density at the centroid
   for 3/s — a clog detector, not a panic detector. Now >3:1 for 1.5/s.
3. **Outflanked** (new, dominant): 8 probes on a `FLANK_RING_R` (22 m)
   ring around the centroid; a sector is hostile only where enemy
   density > `FLANK_DOMINANCE` (1.2×) OWN density (and above the 0.6
   noise floor). The dominance test is load-bearing: take 1 used
   absolute enemy density, and since the ring is SMALLER than a
   1000-man block's footprint (~66 m wide), every line fight read as
   encirclement — units routed within seconds of contact. The largest
   hostile-free arc is the safe side: `flanked = (225° − safe_arc)/225°`
   — a full 3-sector frontal contact scores exactly 0, a pincer ~0.4,
   encirclement 1.0 at `MORALE_FLANKED` (6/s peak). Orientation-free:
   no facing needed, works for any formation shape.
4. **Rout contagion** (unchanged): 2.5/s per routing friendly within
   60 m, cap 7.5.

Then: `pressure × resist / (1 + 0.35 × steady_friends_within_60m
(max 3)) × depletion` — allied support damps ALL psychological terms
(≈×0.5 with 3 friends), kind resist now applies to all of them too, and
the existing depletion factor (fresh regiments shrug, bleeding ones
panic) still multiplies at the end.

## Measured break points (log-verified, BREAKS lines)

| Scenario | Losses at break |
|---|---|
| 5v4 sandbox, frontal light infantry | ~33 % (first break 20 s after contact) |
| 5v4 sandbox, frontal heavies | ~51–59 % |
| Front battle heavies (FL_TEST_FRONT) | ~50–52 % |
| 1-vs-3 envelopment, no support (FL_TEST_ROUT) | ~30 % |
| Surrounded pocket (FL_TEST_SURROUND) | ~28 %, dying 3× faster per capita |

Frontal breaks track the casualty anchor (280 ⇒ ~36 % alone for lights,
~60 % for heavies at 0.6 resist) minus a few points of pressure;
envelopment/encirclement shave more. Acceptances hold: rout test breaks
well before annihilation; surround pocket dies ~3× faster per capita and
BOTH blue detachments end by MORALE BREAK (no stall); the 80-regiment
battle still resolves.

**Pacing retunes (owner rounds 2–3, commits `991f178`, `51c61f4`)**:
`MORALE_CASUALTY` 280 → 200 → **150** ("routing at 500 men left is
shameful"), contagion 2.5 → 2.0/s cap 6.0. At 150: lights break ~67 %
losses from casualties alone; heavies (0.6 resist) CANNOT be broken by
attrition — they hold until flanked or the line collapses. 5v4 after:
first break 31 s after contact at 55 % losses, lights 55–67 %, rallied
regiments fight to 80 %+ before shattering, no heavy broke, battle runs
2.5+ min.

NOTE: if fights STILL feel too short, the remaining lever is kill rate —
FL_COMBAT_SCALE, not morale.

## Knobs

`MORALE_FLANKED` 6.0 (surrounded peak), `FLANK_RING_R` 22 m, `FLANK_T`
0.6 (noise floor), `FLANK_DOMINANCE` 1.2, `MORALE_OUTNUMBERED` 1.5 @
>3:1, `MORALE_SUPPORT` 0.35/friend (max 3). Owner feel pass pending in
the 5v4 sandbox.
