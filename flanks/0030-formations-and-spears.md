# 0030 — Rigid formations, spear infantry, drag orders (2026-07-13)

Branch `feat/formations`, 5 commits (`git log main..feat/formations`).
The owner-agreed big one from the handoff: formations that matter for
battle and morale, M2TW grammar with tweaks. All features log-verified
or screenshot-verified; A/B'd against main where numbers moved.

## What landed

- **Rect rank-and-file formations by default** (`formation.rs`): a slot
  generator writes rotated grid offsets into the existing `units.home`
  column — exactly the seam 0003/0018 designed. One point per regiment
  still; the hot path is untouched. Slots regenerate on EVENTS only
  (order, mode toggle, rally, close-ranks at 6% losses out of contact).
  Greedy crossing-minimizer: sort members front-to-back, chunk into
  ranks, sort each rank laterally. Blob kept (B toggle + FL_TEST_*
  spawns keep their hand-built geometry) for future low-morale levies.
- **Spear infantry** (`KIND_SPEAR`): chainmail + kettle hat + round
  shield, spear modeled VERTICAL on a new PART_SPEAR_ARM (id 4) that
  levels at the enemy in battle stance/spearwall and thrusts on the
  stab (stab-only style). Reach 2.4 m via per-kind scan radius so
  sword units don't pay for it. All shield assemblies moved to
  PART_SHIELD (id 5). Spawn mix FL_SPEAR_FRAC (0.25) between heavies
  and lights. Banners: mid-size banderole.
- **Walls + loose** (keys F / L on the selection): shieldwall (sword
  kinds) and spearwall (spears) = tight files, fronted shields /
  leveled spears, slight crouch, 0.72x advance; loose = 1.9x pitch
  (for the missile future). Combat model: shieldwall absorbs 30% and
  gives up 10% offense; spearwall +25% damage and NULLIFIES the charge
  bonus (CHARGE_MULT moved from swing event to the damage pass where
  the victim's stance is known). Walls damp psych morale terms 0.85.
- **Hold-position vs at-ease** (key H): held regiments never
  auto-engage, never wide-acquire, close only the last step to a
  swing. At ease (default, both teams, ai.rs `auto_engage`): idle
  steady regiments Attack an unbroken enemy inside 40 m, stand down
  when an auto target breaks (no map-edge pursuits) — this also fixed
  the "rallied AI-less sandbox enemies stare" item. Auto orders carry
  a flag so audio UI clicks stay silent (war cries still fire).
- **RMB drag orders + placement preview**: press/drag/release. Drag
  ≥ 5 m paints the front line: width divided by strength, files fill
  each share, front rank ON the line, facing perpendicular AWAY from
  where the selection stands, left-to-right order preserved. While
  dragging: soft green circle per soldier slot (outlines above 3k
  units) + facing arrows. Short click = the old attack/group-move.
  One `line_order` path serves mouse and FL_TEST_FORM.
- **March-in-step + wall poses** via a 4th instance vec4 (location 11:
  march01, wall01, regiment phase — the per-regiment instance channel
  0028 asked for): formed regiments moving under orders walk on one
  shared phase per regiment; contact/charge/rout break step (0.5 s
  per-unit EMA).
- **Charge speed boost 1.15x** during the charge phase (handoff
  wishlist) — also feeds the per-unit SWING_CHARGE momentum predicate.
- **Disorder morale term**: update_groups measures mean slot deviation
  relative to the regiment's own centroid (rigid march = 0; one-tick-
  stale mean-home correction so casualty skew doesn't read as chaos),
  2 s smoothing. Engaged + disorder 2→7 m ramps 0→3.5 morale/s.
  Inspect panel: formation line (files/mob, wall/loose, HOLD) + live
  disorder row.

## Sim changes the acceptances forced (the interesting bugs)

1. **Wall spacing was physically impossible**: slots at 1.0 m but
   separation rest distance is 1.4 m — physics pushed every wall back
   out to normal spacing (nn audit: wall == normal == 1.37). Fix: new
   META_WALL grid bit; same-team pairs BOTH in wall stance use rest
   distance 1.05 (symmetric predicate, no force asymmetry). Wall pitch
   set to match. After: nn wall 1.06 / normal 1.37 / loose 2.13.
2. **Ranks never finished dressing**: hold-at-anchor deadzone (1.5 m)
   + gain 0.4 were tuned for jittered spawn homes. Rigid slots sit AT
   the separation rest distance (force-free when dressed), so deadzone
   0.7 / gain 0.6 is stable: slot error 0.6–0.8 m after settling,
   move-avg audit still 0.000 (no twitch regression).
3. **spatial META kind was ONE bit** and the mass lookup booleanized
   it — a third kind would have silently aliased into kind 1. Now a
   2-bit field read via `meta_kind()`.

## Verification (all on this branch, owner doctrine)

- FL_TEST_ROUT: blue BREAKS at 513/1000 (the ~50% band), flees, 489
  despawn as fled.
- FL_TEST_SURROUND: pocket annihilated 0/500 vs line 114/500, both
  blue detachments end by MORALE BREAK.
- FL_TEST_ORDERS (200k): long-march spread ratio 1.31 (< 1.5), ZERO
  charge/engage events. Stage-3 group-move report doesn't fire within
  320 s — identical on main (one straggler regiment keeps its order;
  pre-existing test-duration quirk, not a formations regression).
- FL_TEST_FRONT (200k engaged): drift 2.40 m vs main 2.47 m (same);
  nn 0.68/0.82 vs main 0.69/0.82. Step mean 7.3 ms vs main 6.2 ms —
  ~+1.1 ms, mostly the 25% spear regiments' wider reach scans; worst
  spike 18 vs 15 ms; 33 ms budget fine.
- NEW FL_TEST_FORM: line_order → files 19 each, facing 0.00; settled
  slot err 0.62–0.77 m OK; spacing nn ordering OK (numbers above).
- 200k default battle (AI + at-ease live): no panics, step mean
  6.2 ms, sync 2.4 ms, 8 at-ease engagements.
- Screenshots: spearman mesh close-up, spearwall hedge (panel:
  47 files SPEARWALL, psych x0.72), shieldwall front (psych x0.51),
  mid-drag preview (3k green slot circles at 172 fps), post-release
  line march. WGSL greps clean on every run (0028 rule).

## Notes for the owner playtest

- Drag facing rule: perpendicular to the line, AWAY from where the
  selection currently stands (not drag-direction chirality — camera-
  independent and never faces troops back at themselves). About-face =
  draw the line behind the regiment.
- Narrow drags make deep columns by design (files = width/pitch).
- Keys: F wall, L loose, B blob/ranks, H hold, Backspace halt
  (unchanged), G viz, R restart.
- Loose keeps current files → the block gets much deeper; a "keep
  frontage, halve depth" variant is a candidate tweak.
- Tuning knobs all const-named in movement.rs (WALL_*, CHARGE_*) and
  morale.rs (MORALE_DISORDER, DISORDER_FREE/FULL, WALL_STEADY).

## Deferred / next

- Wall vs charge counter-damage (points punishing the charger) if the
  nullify alone feels weak.
- Loose-order missile damage reduction — with archers.
- Column march / wheeling as explicit formation moves.
- orders.rs split done for selection; order-issuing could still split
  from Groups data if it grows again.
- FL_TEST_ORDERS stage-3 straggler (pre-existing): the group-move
  arrangement metric never logs; worth a look on the refactor branch.
