# 0005 — Art direction locked + terrain relief overhaul (2026-07-08)

## Direction

Owner verdict: the **War of Dots / Minecraft-like vibe is the direction** —
chunky flat-shaded low-poly, hard color bands, readable masses of small units
over deformable ground. Locked in; future visual work builds on this, not
toward realism.

Second verdict: first-pass terrain was too flat — "basically flat", the
environment needs to feel dynamic.

## Relief overhaul

Generation is now three layers (all hash-based value-noise FBM, no deps):

- **Large landforms**: `fbm(p/320) * 22` — rolling ~40 m-wavelength hills
- **Medium detail**: `fbm(p/90) * 4.5` — local undulation the armies react to
- **Ridged peaks**: `(1 − |fbm(p/260)|)² * 16` — sharp ridge lines / massifs

Center softening instead of flattening: multiplier `0.35 + 0.65·t` beyond
r=140 (was 0.25 starting at r=220) — the battlefield itself rolls now, and
the outskirts get real mountains (~35–45 m) with **snowcaps** (new top color
band > 24 m, rock band 17–24 m). +2.6 m base bias keeps the midfield in
grass; dirt only in genuine hollows and crater floors. Sun lowered
(pitch −1.1 → −0.75 rad) — flat-shaded relief lives on directional contrast.

Result: armies drape visibly over contours, stream through valleys between
brown highlands, snow ridges on the horizon. Steep-face scree threshold
raised (0.55 → 0.75 slope) so hillsides keep their band colors and only real
cliffs go gray.

## Owner feedback (post-overhaul)

"Still not enough, but better for now" — terrain drama needs another pass
later (bigger amplitude? cliffs/plateaus? canyons?). Not blocking milestones;
revisit after M6 or when terrain becomes tactically relevant (M5 fronts).

## Numbers unchanged

80–195 fps depending on view, sim tick ~2.3 + ~3.7 ms. NN spacing actually
loosened (avg 0.82–0.86) — slope penalties spread the crowd. Craters +
remesh unaffected.
