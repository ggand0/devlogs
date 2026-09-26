# 0039 — Rebuild on the evidence rule; the press packing was the seal

Date: 2026-07-18. Branch: feat/m2tw-melee (fresh from main 0a3de56).
Owner rulings this session: reproduce M2TW, implement NOTHING without
M2TW evidence; the old branches deleted (preserved: backup/
rank-discipline e4f0020, backup/melee-brawl 20f06a7, full bundle
work/backups/cascade-full-2026-07-18.bundle); back up BEFORE any git surgery,
always. Local main was 27 commits stale (pre-formations) and was
fast-forwarded to origin/main 0a3de56.

## What this branch is

Cherry-picks of the four evidenced commits (17e9743 freeze/reform
model, 811bd92 diag, 30fbeee body rest + destination freeze, 9ef8124
jam at body scale) plus a5be67e REMOVING the press packing
(META_PRESS bit, press flags, tight same-team rest for fighters).

Evidence ledger, per mechanism kept:
- Destination freeze: descr_pathfinding.txt formation_hold_distance
  20.0 (extracted from the owner's install); path computed once.
- Body-contact rest ENEMY_SEP_RADIUS 0.95: Feral EDU docs — hidden
  radius default 0.4, collision separate from formation spacing;
  M2TW rest/pitch 0.8/1.2 = 0.67, ours 0.95/1.4 = 0.68.
- Reform as discrete state + slot memory: M2TWEOP engine structs
  ("reforming" action state; formationX/Y kept alive).
- NO packing rule for fighting ranks (the removal): vanilla M2TW
  keeps/loosens its grid in melee and the engaged units slowly
  interpenetrate (strong community record; Stainless Steel modded it
  tighter, i.e. vanilla is looser). The META_PRESS 1.05 rest had no
  source and SEALED the seams: pressed fronts left ~1.05 m gaps for
  0.95 m bodies, so symmetric fights collapsed to a two-rank duel
  line, and friendly regiments attacking one target packed into each
  other (the blob). Both owner-rejected defects, one un-evidenced
  mechanism.
- Jam/crowd yield: our substrate's stability control (maps loosely
  to the engine's crowded/sidestep gating), not an M2TW claim.

## Measured (this branch)

- SYMMETRIC 2v2 (the owner's scenario, first time instrumented):
  developed fight shows reach across SIX rank bins — e.g. r0 96%,
  r1 97%, r2 73%, r3 66%, r4 59%, r5 43%; windup 5–34% throughout.
  The two-rank duel line is gone; the band is deep and both sides
  interleave. nn 0.72/1.00 during the fight, move avg 0.017 (settled
  0.002–0.003, marching 0.144).
- PILE: band intact (r0–r3 at 94–100%, r4 78%).
- Gate-off DIR: digit-identical to baseline yet again (475/300/619,
  22.2/28.7/49.6, control 103, yaw 0.00).

## Pending

Owner feel pass — twitch, blob, band depth, pacing are his verdicts.
Not re-run yet at this tip: CHARGE/ROUT/SURROUND/FORM bands, 200k
disorder + perf (previous tip's numbers in 0038; press removal
should only loosen fighting crowds).
