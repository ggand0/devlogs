# 0015 — 200k scale test: no perf wall (2026-07-09)

Owner direction: M2TW-style field battles first (flow-field/sieges later),
but confirm the engine holds >100k units before building features on it.
Added `FL_UNITS` (units per team, default 50k, runtime env knob — commit
`cf29061`) and ran 200k total through idle and full-engagement scenarios.

## Numbers (opt-dev, RTX 3090 / 3900X; trust sim + GPU times, not FPS)

| Scenario | grid | step | sim total (33.3 ms budget) | transparent pass |
|---|---|---|---|---|
| 100k idle (baseline) | 2.7–3.5 ms | 2.0–2.2 ms | ~5.5 ms | 0.57 ms |
| 200k idle | 4.0 ms | 3.3 ms | 7.3 ms | 1.29 ms |
| 200k full engagement | 4.5–4.7 ms | 5.8–6.1 ms | ~10.6 ms | 1.52–1.54 ms |

- Engagement test: FL_TEST_FRONT with 100k/side; ~37k dead by t+45 s,
  front-line contour drawing across the whole battle line, both instance
  buckets correct throughout (fix from 0014 exercised at 2× scale).
- Sim scales sublinearly-ish in this range (2× units → ~1.9× sim tick in
  engagement); worst case uses ~32% of the fixed-tick budget.
- Render: 2 instanced draws, ~1.5 ms transparent pass at 130k drawn.
  Frustum culling keeps drawn counts at 60–130k depending on view.
- Uncontended FPS peaked 250 at 200k; on-screen FPS is compositor-clamped
  as always (devlog 0013).

## Verdict

No performance issues at 200k. The deferred perf backlog (SIMD separation,
parallel grid rebuild, GPU-driven culling) stays deferred — revisit around
300k+ or when regiments/swing combat add sim cost. Spawn formation caps at
~125k/team before rows hit the terrain edge (noted in `units_per_team()`).

## Next

All feature work (unit types + facing, swing-timer combat, regiments)
proceeds on the m2tw feature branch per owner call. Field battles first;
flow-field pathfinding + sieges deferred.
