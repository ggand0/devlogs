# 0038 — The brawl band: body-contact rest + destination freeze

Date: 2026-07-18. Branch: feat/melee-brawl (cut from e4f0020 with
owner approval; feat/rank-discipline retired). Design approved by
owner in plain language before coding. Full plan + evidence record:
docs/plans/melee-brawl-band.md. Gate: FL_RECTFIGHT=1.

## The two-legged root cause (both log-verified this session)

FL_DIAG_MELEE (built last session, never run) finally ran. Baseline
gated PILE, 23 samples over 100 s: r0 at 2.0–3.7 m mean from the
nearest enemy (20–64% in reach), r1 nearly always 0%, r2+ 0% in
EVERY sample. Two legs:

1. Man level: cross-team separation rested every enemy pair at
   SEP_RADIUS 1.4 = our formation pitch. Gap-to-body ratio 1.0 — an
   enemy block had no gap a body could enter. M2TW (data extracted
   from the owner's install + Feral docs): bodies rest ~0.8 m inside
   a 1.2 m grid, ratio 1.5, enemies flow into the grid's gaps.
2. Regiment level: the FREEZE parked the frame where the block stood
   at first wind-up (one corner touching). Every man without an
   enemy inside the 4 m acquire radius was pulled to a parked slot.
   Measured intermediate build (body rest alone, no freeze fix): r1
   peaks 38%, r2+ still 0% — necessary but not sufficient.

## The model (smallest diff, all under FL_RECTFIGHT)

- movement.rs: ENEMY_SEP_RADIUS 0.95 — cross-team pairs rest at body
  contact (ratio 1.47 ≈ M2TW's 1.5). Wide-acquire and the memo-close
  stay open until CROWD_STOP instead of CROWD_SLOW; the memo-close
  surge yields (1 − jam) — steering brakes on genuine body-pack,
  in-reach combat execution keeps its bypass.
- frontline.rs: freeze the DESTINATION, not the block. On the engage
  edge an attack order's anchor snaps once to the TARGET's centroid
  at contact (M2TW: path computed once + formation_hold_distance);
  men keep walking into the enemy mass and bodies stop them at the
  interface — surplus ranks stacking behind IS the press. Defenders
  anchor where they stand.
- formation.rs: engaged close-ranks no longer retreats the anchor to
  the current centroid (the destination must not walk back; a shoved
  defender leans back toward his held line).
- Committed the FL_DIAG_MELEE logger (regiments.rs).

## Measured (per-rank profile, gated PILE)

r0 d0.79–1.07 reach 93–100%, r1 d0.79–1.23 reach 96–100%, r2
d1.00–1.46 reach 85–100%, r3 d1.27–1.89 reach 55–96%, r4 14–74%,
r5 2–8%; wind-up spread over five ranks (peaks 24–35% in r2–r3).
The band is 4–5 ranks deep per side and the fight happens at body
contact. nn min 0.73–0.76 / avg ~1.00 (cube width 0.62 — no visual
overlap in PILE), step ~0.9 ms, zero spike lines.

## Battery

- Gate-off DIR: final accumulators digit-identical to the 0037
  baseline (475/300/619, 22.2/28.7/49.6, control 103, yaw 0.00).
- DIR gated: rear per-hit 50.2/23.0 = 2.18x (band 2.19x), yaw dev
  0.00. Kill MIX went front-heavy (591/348/439 vs rear-dominant
  before): front fights now finish victims faster. Per-hit direction
  mechanics intact; mix is pacing.
- CHARGE gated: while both lanes stand, wall dz −1.5 m vs open
  −2.7 m; wall lane kills 236 heavies vs open 121 (~2x); wall spears
  outlive open through the fight (216 vs 182 at t=36). NEW: both
  spear lanes eventually break and are run down — a 500-heavy charge
  now decides against 500 spears. Wall-beats-open ordering intact;
  absolute outcome is the new pace.
- ROUT: breaks at ~50% casualties (497→429 alive of 1000 at ROUTING
  onset) — the same organic morale band; morale.rs untouched.
- SURROUND: pocket dead by t=40, line holds 227/500 to t=60 —
  pocket-dies-faster criterion holds strongly (both fights finish
  inside the window at the new pace).
- FORM gated: spacing nn wall 1.06 < normal 1.37 < loose 1.80
  (exact 0037 values); slot err 0.68/0.71/0.75 on regs 1–3, reg 0 at
  0.91 (test self-reports OK); facing 0.00–0.02.
- PILE: victim 500→188 in 24 s of contact, breaks ~62%, wiped by
  t=76 including the rout chase. Old build: ~33% at t=100.
- 200k (FL_UNITS=100000 FL_ENEMY_REGS=100 FL_ARMY_GAP=60): 111–118
  regiments engaged (was 85–93 — more of the army fights). Disorder
  climbs 1.1→8.1 by t=96, 8.1–10.5 through t=157: the metric now
  measures PRESS DEPTH (slots are deliberately deep in the enemy),
  not parked-block smear; still well under the 13.7-and-climbing
  ungated blob. Needs the owner's eyes for band-vs-smear judgment.

## Known costs (owner rules)

1. Pacing is much faster everywhere (4–5 ranks swing instead of a
   fraction of one). Honest levers if too fast: BASE_DMG, cooldowns.
2. 200k perf: step spikes to 9.7–25.4 ms in the deepest crush (was
   7.0–7.8), ~2 spike lines/s, grid 4–5 ms; body-packed cells put
   ~2x candidates in every neighbor scan. Levers: ENEMY_SEP_RADIUS
   up a touch, scan work caps, or accept.
3. 200k nn min dips to 0.49–0.56 in the deepest crush (PILE stays
   0.73+) — transient body overlap can be visible there. Levers:
   CORR_GAIN/CORR_MAX, ENEMY_SEP_RADIUS.
4. Late-battle disorder creeps (9.7→10.5 t=136–157) — bounded ~half
   a regiment depth so far; watch in longer battles.

## Owner feel pass #1 (regressions found, one fixed)

Owner verdict on the first build: rear-rank TWITCH (the old solved
problem, back), attackers still BLOB on a shared target, difference
vs main not legible. Twitch root cause: the jam density was keyed to
the tight pair rest radii, so at full body-pack the crowd sum peaked
~0.9 — below CROWD_SLOW 1.2 — and the brake + viscous damping were
mathematically DEAD under the gate; the press oscillated against
undamped springs. Fixed (20f06a7): jam density reads at the original
SEP_RADIUS body scale for every pair, ungated arithmetic unchanged.
Measured: mid-press move avg roughly halved (0.047/0.025/0.019/0.013
vs 0.065/0.041/0.035/0.032; settled floor 0.004), nn min 0.73 ->
0.76-0.79, band retained and deeper (r0-r3 100% in one sample).
Further lever if his eyes still see twitch: CROWD_STOP under the
gate (body-pack tops out ~1.9-2.1 on the fixed scale, so 2.5 is
beyond-physical there).

Blob diagnosis (fix pending owner ruling, evidence in the reply +
plan doc): (a) every attacker freezes its destination on the SAME
target centroid; (b) META_PRESS packs tight against ANY same-team
man — two attacking regiments dissolve into each other. M2TW keeps
friendly units as distinct blocks (movement-level blocking structs;
"two blocks in contact, not a merged mass"); enemy-vs-enemy
interpenetration at the interface is vanilla, friendly merging is
not.

## Not done / next

- Owner feel pass (the only verdict that counts).
- Research caveat: the adversarial verify pass over the M2TW
  behavioral claims died on the spend limit; claims rest on primary
  sources (Feral docs verbatim, M2TWEOP structs, extracted game
  data) + last session's 8 verified claims.
- Formation depth (22 ranks vs M2TW's 3–5 by EDU data) raised with
  the owner as a separate gameplay lever; no action.
