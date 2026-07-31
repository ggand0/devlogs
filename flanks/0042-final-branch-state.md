# 0042 — Final branch state: feat/m2tw-melee at bac60e2

Date: 2026-07-20. Six commits off main 0a3de56, all gated
FL_RECTFIGHT=1.

## What the branch does vs main

Order layer (the surviving improvements):
- Attack destination freezes once the ordered fight becomes real
  (percentage-gated: 3% of living strength, floor 4). A trickle of
  overflow duels never halts a march.
- Attack approaches route around friendly formed blocks.
- Broken/Move-ordered men pass through friendly lines at body scale.
- Pursuit limited to 120 m from the fight origin (M2TW:
  attack-dist-multiplier 3.0 × max-engage-dist 40, from
  config_ai_battle.xml).
- Regiment re-forms on disengage; engaged close-ranks feeds reserves
  forward on casualties.

Combat spacing: ENEMY_SEP_RADIUS = 1.4 (same as main). The owner
tested 0.95, 1.15, and 1.2 — all produced visible weapon clipping
with the current model scale. The multi-rank band needs model
proportion work before the rest distance can drop.

HP bumped 1.5x (heavy 240, light 135, spear 150) as a temporary
pacing placeholder. The owner plans to rework the morale system to
control pacing properly.

## Full regression battery (this commit)

- DIR gate-off: 475/349/568 (kill distribution shifted by the HP
  change — soldiers survive longer and take hits from more
  directions; per-hit damage 22.4/29.2/49.0 within jitter of
  baseline 22.2/28.7/49.6; combat model unchanged)
- DIR gated: 511/310/594, per-hit 22.6/30.0/49.1, yaw 0.00
- CHARGE: wall dz −1.5 m vs open −3.6 m, wall kills more chargers
  (162 vs 126) — wall-beats-open ordering intact
- ROUT: breaks at ~49% (508/1000 alive) — the ~50% band holds
- SURROUND: pocket dead by t=40, line holds 253/500 — criterion
  solid
- FORM: all four OK, spacing wall 1.05 < normal 1.36 < loose 1.87
- 200k: disorder 6.6–9.9 m (plateaus, never runs away), 53–90
  regiments engaged, step spikes 21.7 ms, nn 0.67–0.70

## Open items (documented for future work)

- Multi-rank fighting band: needs model scale/proportion work so
  ENEMY_SEP_RADIUS can drop below 1.4 without weapon clipping
- Morale rework: owner's project, will replace the 1.5x HP
  placeholder
- Fighting withdrawal: frozen anchor goes stale if a locked enemy
  retreats slowly
- Interposed enemy: fighting a regiment that isn't the ordered
  target doesn't freeze the path
- Pursuit range: currently 120 m flat; the engine may scale it per
  unit type (cavalry longer)
