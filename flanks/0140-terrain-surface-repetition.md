# Terrain surface repetition
Written by GPT-6 Astra.

Date: 2026-09-26. Branch: `feat/terrain-look`. Commit: `c9303f3`, Reduce terrain repetition across viewing distances.

Isolated v1 lighting, texture and coverage to confirm fine repeating texture rows and boxy ground color regions. Replaced broad coverage with one terrain-aware map; prepared neutral filtered grass detail; added arbitrary sample rotations and variance correction; separated turf-scale variation from close texture detail with distance/footprint fades. The first v2 candidate was too smooth, so the turf scale restores intermediate detail.

Build, strict clippy, both existing terrain tests and changed KTX2 validation pass. Continuous zoom/orbit and crater recordings were checked in consecutive sampled frames. Added continuous motion as a negative period on the existing camera sweep variable. Heights, map dimensions and sim methods remain unchanged; no sim hash was needed under the current rendering-only rule.

Review page: `tmp/shots/terrain-surface-v2/index.html`. Detailed implementation and GPU timing limits: `work/handoffs/HANDOFF-terrain-surface-v2-2026-09-26.md`. A short 200k-soldier gameplay check at recorded 1935 MHz gives opaque-pass times of 0.10 ms plain PBR and 0.34 ms surface. Close and wide clocks vary; do not claim all-view matched full-clock validation. All surface samples were below 0.8 ms. Ten crater rebuild samples took 0.08 to 0.27 ms for one to four chunks.

No push or PR. River redesign, larger terrain and environment remain subsequent stages. No shared source or main-tree code edits.
