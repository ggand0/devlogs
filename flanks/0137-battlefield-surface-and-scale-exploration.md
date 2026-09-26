# Battlefield surface and scale exploration
Written by GPT-6 Astra.

Date: 2026-09-26.

Read the parallel-worktree instructions and the terrain-look brief, inspected the grassland reference, and traced terrain, vegetation, water, camera, deployment, spatial grid and unit lighting in `flanks-gfx` at `fde9a0c`. The graphics branch is `feat/terrain-look`, initially equal to local `main`.

Proposal: `assets_dev/terrain/proposal_v1/proposal.md`. This records the requested code exploration, not a completed in-engine prototype comparison. No source edits, builds, game launches, Blender runs or performance measurements.

Recommend natural pasture materials and smooth terrain normals first, followed by a separate 2,048 × 1,536 m battlefield layout. Current 1,024 × 768 m terrain has 393,216 triangles. Default armies fill nearly the whole width with a 60 m gap, so size work must also reconsider deployment. The dense collision grid scales with soldier bounding area and caps each axis near 3,072 m; consult the concurrent performance work before larger fields. Unit illumination is hardcoded separately from the scene light, which matters for the later environment lighting pass.

The proposal ranks visual problems, describes the intended composition and surface approach, and records geometry cost calculations, simulation implications, source boundaries and staged review points. Poly Haven's CC0 policy was verified; no assets were downloaded. No commits.
