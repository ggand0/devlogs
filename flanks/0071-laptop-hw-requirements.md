# 0071 — hardware: laptop requirements for 50k-100k (travel machine research)

**Date:** 2026-07-31
**Branch:** n/a (research only, no code)
**Context:** traveling from 2026-08-01; picking a temporary laptop
(3-4 weeks) for debugging FLANKS + viewskater dev. Candidate:
Lenovo IdeaPad Slim 5 Gen 10 (16AKP10). NOT COMMITTED — personal.

## Where the compute actually goes (from devlogs 0047/0049/0050)

Since the pipelined tick (0050), the sim runs on a dedicated worker
thread at 30 Hz with a 33 ms budget, fully off the render path. On
the dev box (Ryzen 9 3900X, 12C/24T Zen 2 + RTX 3090):

- 100k tick: p50 3.7 / p90 4.6 / max 7.2 ms — ~5x headroom.
- 200k tick: 6-9 ms.
- Remaining hitch class is render-side: render-world CPU bursts at
  far zoom (25-29 ms — bevy batching + instance prep, leans on
  single-core), and at 200k raw render throughput.
- GPU pass itself was ~1.4 ms on the 3090 — GPU demand is modest by
  dGPU standards; an iGPU is ~10-15x slower and shares memory
  bandwidth with the CPU.

**Laptop requirement profile for smooth 50k-100k:** ~8 modern cores
(Zen 4/5, M2 Pro class, Core Ultra H) for the sim, strong
single-core for bevy's render world, GPU at Radeon 780M/860M level
or better at ~1200p. Existing data point: M1 MacBook (late 2020,
4P+4E, ~2.6 TFLOPS GPU) holds 20k at decent fps but runs hot —
consistent with the profile.

## IdeaPad Slim 5 Gen 10 (16AKP10) verdict

Specs: Ryzen AI 7 350 (8C/16T, Zen 5 + Zen 5c), Radeon 860M
(8 CU RDNA 3.5), DDR5-5600, 16" 1920x1200 IPS, 60 Wh.
Benchmarks: multi-core ~35% below the 3900X; Zen 5 single-core
well ahead of Zen 2 — exactly the shape this game wants.

- **Sim:** 100k ticks estimated 6-12 ms — comfortably inside the
  33 ms budget. Not the limiting factor.
- **50k:** comfortable, near 60 fps at native 1200p.
- **100k:** playable; expect 30-45 fps dips at far zoom over a
  dense field (GPU/bandwidth-bound, not sim-bound).
- **200k:** no.
- Caveats: run plugged in on performance mode (28W-class chip drops
  hard on battery, and power management noise pollutes frame-time
  measurements — see the gnome-shell lesson, 0051). Prefer the
  32GB config: iGPU eats system RAM. viewskater / wgpu apps are far
  below this workload — trivially fine.
- **Does NOT transfer:** absolute render numbers. Far-zoom render
  tuning measured on the 860M won't predict 3090 behavior —
  directional only. Sim-side work (hitch attribution, FL_HASH
  determinism, FL_CAM_TOUR runs) is hardware-independent and
  transfers fully.

If a gaming laptop were ever wanted instead: any RTX 4060/5060
machine (Lenovo LOQ / Legion 5 + Zen 5, or ASUS TUF A14) restores
the desktop-3090 experience including smooth 200k — the game's GPU
demand is small; the 3090 was never the bottleneck.

## Side benefit

The 860M run doubles as a real low-end target data point for the
0.1.0 itch release — iGPU laptop players are exactly the audience
for a free prototype. First check on the new machine:
`FL_UNITS=50000 cargo run --profile opt-dev`, grep `[hitch]` —
`main` high + `tick total` low = render-bound, as expected.

## Sources

- Lenovo PSREF: https://psref.lenovo.com/syspool/Sys/PDF/IdeaPad/IdeaPad_Slim_5_16AKP10/IdeaPad_Slim_5_16AKP10_Spec.pdf
- LaptopMedia specs: https://laptopmedia.com/laptop-specs/lenovo-ideapad-slim-5-389/
- CPU comparison: https://versus.com/en/amd-ryzen-9-3900x-vs-amd-ryzen-ai-7-350
