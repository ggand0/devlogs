# 0019 — Soldier readability, sword-swing animation, morale retune (2026-07-10)

Owner feedback on the MVP: units read as "escape-sign icons", swings were
just the victim's white blink, and regiments routed too easily. All three
addressed in `9d29b35`.

## The readability insight: color separation, not silhouette

First attempt improved the silhouettes and it still looked like boxes —
because ONE flat team color collapses any silhouette into a mass. The
Thronefall / Kingdoms-and-Castles look comes from per-part materials:
skin, steel, wood, blade, team cloth. Implemented as per-vertex colors in
the SHARED mesh (zero per-instance cost): rgb = material, alpha = how much
the per-instance team color blends in. Palette is a 7-entry const table in
`unit_meshes.rs` — tuning the look is one-liners.

Rules learned iterating on screenshots:
- Team cloth must DOMINATE; steel darker than cloth and team-tinted
  (a=0.35) or knight masses turn gray and armies stop reading.
- Bright fixed-color blades read at any distance; they double as the
  combat-activity indicator across a battlefield.
- Mesh vertex color = shader location 5 in bevy → instance attributes
  moved to locations 8–10.

Knight: tapered cuirass, pauldrons, full helm + nose guard, kite shield,
arming sword — 204 tris. Man-at-arms: tunic, kettle hat over bare face,
buckler, short sword — 192 tris. GPU cost of ~2× triangles: transparent
pass ~3 → ~3.5-4 ms wide-view, nothing at gameplay zooms.

## Swing animation: part rotation in the vertex shader

The UV channel (units are untextured) carries `[part_id, pivot_y]` per
vertex. The shader pitches parts around their pivot:
- Sword arm: raises up/back through the wind-up (`lunge` 0→0.8), then a
  fast chop over the last 15% — the blade lands exactly on the tick the
  damage event fires. Light walk sway when idle.
- Legs: opposite-phase walk swing driven by the existing move amount +
  per-unit seed.
No new per-instance data, no skeleton — a few vertex-shader rotations.

## Swing logic (owner asked): no hitbox

Target-lock duels, not swept volumes: Ready picks the nearest living
enemy within reach (1.8/2.0 m circle, from the same grid scan as
collision), WindUp plants feet and faces it, the strike validates (alive,
in reach×1.15) → damage, else whiff. One victim per swing, ever;
"surrounded dies faster" = being locked by many attackers at once.

## Morale retune: pressure scales with depletion

Root cause of "routs too easily": outnumbered (−5/s) and rout-contagion
(−18/s cap) drains ignored regiment health — a FULL-STRENGTH regiment
near three routers broke in ~4 s having lost nobody; one break cascaded
the whole army (hands-off run: 99/100 broken at 14% losses).

Fix: `pressure × depletion` where depletion = clamp(1.4 − 1.2·frac,
0.15, 1.4) — fresh regiments shrug off psychology (0.2×), bleeding ones
panic (>1×). Coefficients also softened (outnumbered 3/s, contagion
2.5/s cap 7.5). Casualty morale unchanged: ~35% losses alone still
breaks (FL_TEST_ROUT: breaks at 666/1000, same as before).

Result (hands-off AI battle): breaks now happen at ~25–35% real
casualties; at t=150 s the armies were at 21–27% losses with a coherent
fighting front and visible desaturated rout streams — hold → bleed →
break → cascade, in that order.

## Misc

- `FL_CAM_PITCH` env knob (with FL_CAM_DIST) for low-angle screenshot
  verification.
- Screenshot workflow note: match windows by PID; the owner play-tests
  live and a name-matched `xdotool` grab can capture HIS window.
