# 0026 — Regiment inspect panel + attack-hover highlight (2026-07-12)

Commit `93d672a` (+ `991f178` readout plumbing) on `feat/battle-feel`.
Owner: "I should understand more while playing" — a TW-style plaque with
the unit's morale ± factors, paired with the attack-order hover preview.

## Hover (`orders.rs::Hover`, updated per frame)

`Hover { enemy, own }` = regiments under the cursor, found with the SAME
`regiment_at` pick logic (20 m `ATTACK_PICK_RADIUS`) as the right-click.
That identity is the point: `hover.enemy` IS what RMB would attack, so
the red tint can never lie. Consumers:

- `render_units.rs` sync: hovered enemy regiment's units tint red
  (priority after broken-desat and selection-yellow) — attack preview.
- The inspect panel below.
- `issue_order` still does its own pick at click time (no cross-system
  ordering dependency; one frame of staleness in the visuals is fine).

## Inspect panel (`overlay.rs`)

Bottom-right dark plaque, visible when hovering any regiment (enemy
first, matching the attack context), falling back to a lone selected
regiment. Shows: kind + regiment id + team, strength `x/y men`, morale
0–100, state (STEADY / STEADY - engaged / STEADY - CHARGING / ROUTING /
SHATTERED), then the live drain rows in morale/s:

```
Heavy Knights 1 (blue)
773/1000 men    morale  70    STEADY - engaged
casualties      -2.4/s
flanked  40%    -0.5/s
outnumbered     -0.0/s
rout nearby     -0.0/s
allies x1   psych x0.44   depletion x0.47
```

Data comes from `regiments.rs::MoraleReadout`, filled by `update_morale`
each tick: per-second rates AFTER all multipliers (so rows sum to the
actual drain), casualty rate smoothed over ~1 s (per-tick spikes are
unreadable), plus the flanked fraction, ally count, psych multiplier
(resist / support), depletion, and a recovering flag.

Verified live: two screenshots of the same fighting regiment seconds
apart show morale 70 → 61 and flanked 40 % → 20 % tracking the actual
salient shape. Enemy red tint shares the (verified) selection-tint code
path; owner eyeballs it on first mouse-over.

## Next hooks

- Attack marker + hover tint could merge visual language (marker on
  hover as preview, not only after the click).
- The panel is the natural home for future TW-style flags: general
  nearby, uphill, tired, and the eventual unit-name/voice-line pass
  (plan section 5).
