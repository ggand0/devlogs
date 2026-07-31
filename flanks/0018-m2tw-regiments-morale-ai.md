# 0018 — m2tw branch: regiments, morale/rout, skirmish AI (2026-07-09)

MVP milestones 3–5 (plan `docs/plans/m2tw-mvp.md`; commits `0e34e80`,
`984c6ad`, `263b500` on `feat/m2tw`). The game is now a playable M2TW-lite
skirmish: two armies of 100 regiments each, heavy/light infantry, swing
combat, morale-driven collapse, and an AI opponent that attacks on its own.

## Regiments (`0e34e80`)

- `regiments.rs` spawns both armies as a FIXED list of regiments
  (`FL_REG_SIZE`, default 1000; `FL_HEAVY_FRAC` of regiments are heavy, in
  the front ranks). Layout auto-fits the terrain — rank width compresses
  instead of spawning past the edge (first attempt squashed back rows onto
  the bounds clamp; the nn audit caught it as nn_min = 0.00).
- **Cohesion without a steering field**: a regiment order is ONE point,
  rigidly translated per unit by its spawn-captured `home` offset. The
  block MOVES, it never converges. No order = hold at the anchor (a
  standing order with deadzone + reduced gain). Rigid formations later ==
  slot generator writing the same `home` column.
  - Consequence discovered by test: a single group can never converge on a
    point (an annulus ordered to its center doesn't move). Encirclement is
    multiple regiments ordered inward — which is the correct game grammar.
- Selection is per-REGIMENT (stroke hits promote at max(8, 3%) of
  strength; C-cut and split_selection deleted — regiments are permanent).
  RMB translates the target per regiment: group moves preserve the army's
  arrangement.
- Verified: 350 m long-march keeps RMS spread ratio 1.23; a held regiment
  near the front drifts 0.38 m in 20 s (the 0012 drift complaint is fully
  dead); surround acceptance still holds after re-plumbing (pocket
  annihilated vs 55% losses in the frontal control).

## Morale & rout (`984c6ad`) — battles END now

- Per-regiment morale drained by fresh casualties (~35% losses break),
  local >2:1 density ratio, and routing friends within 60 m; slow recovery
  when unengaged. Heavies resist (0.6×).
- Break → Routing: uncontrollable, full-speed flight to the own map edge,
  no attack initiation (pursuers hit for free), desaturated rendering.
  Rally roll after 8 s when clear of enemies (back at morale 40); below
  15% strength → Shattered, never rallies. Reaching the edge despawns as
  `fled` (separate stat from kills).
- FL_TEST_ROUT: 1v3 — blue breaks at 666/1000 alive, flees, despawns.
- 200k FL_TEST_FRONT at FL_COMBAT_SCALE=4 **terminates decisively in ~5
  minutes**: 395 breaks, 200 rallies, 195 shatters, final 2,966 blue vs 0
  orange. The M6 endless-endgame tail is gone.

## Skirmish AI + outcome (`263b500`)

- `ai.rs`: idle Steady enemy regiments attack-move onto player regiments,
  closest-first with a +40 m penalty per already-committed attacker
  (greedy spread → the AI forms a line, not a pile). Re-targets when the
  victim dies or breaks. FL_AI=0 or any FL_TEST_* disables.
- Victory banner when a side has no Steady regiment left.
- Hands-off run: AI advanced, enveloped the PASSIVE player army, broke it
  in ~90 s (99 regiments broken, 66k fled → DEFEAT banner). Passive
  defense loses; the player has to actually play.

## Perf state @200k engagement

grid 1.6–2.3 | step 5.5–7.6 (worst during regiment battles — the SIMD
kernel, PR6, is next if more headroom is wanted) | field ~1 | audit ~2 |
sync 1.5–3 | GPU transparent ~3 ms. No hitches. Tick worst ≈ 12 ms of the
33.3 budget.

## MVP status

PR1–PR5 of the plan are DONE — the MVP loop (deploy → order regiments →
swing melee along a line → morale collapse → victory/defeat) is playable.
Remaining on the branch: PR6 SIMD kernel (stretch), then the rigid-
formation milestone. Owner playtest is the real acceptance now.
