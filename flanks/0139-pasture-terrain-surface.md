# Pasture terrain surface
Written by GPT-6 Astra.

Date: 2026-09-26. Branch: `feat/terrain-look`.

Implemented the first surface stage of the terrain proposal: indexed meshes with smooth shared normals, pasture/soil/stone material blending, three Poly Haven CC0 texture sets, mipmapped KTX2 assets, stochastic grass sampling, and crater exposure with normal updates across chunk boundaries. Terrain heights, movement/blocking/wading rules and map dimensions are unchanged. Source changes are limited to terrain.rs and new terrain assets/shader files.

Mesh vertices: 1,179,648 to 209,088. Triangles: 393,216 unchanged. Mesh attribute/index storage: 45 MiB to 12.48 MiB per copy. The six texture files occupy 23,232,676 bytes on disk and about 32 MiB of GPU texture memory including mips. Sources, licenses and an offline preparation script are included beside the textures.

Build and strict clippy pass in opt-dev. Both geometry/crater tests pass. All six KTX2 files pass Khronos validation. Captured and inspected gameplay, close/wide camera switches, crater updates, the river variant, and a 200,000-soldier battle. The five final runtime logs contain no shader/asset errors or panics. No matched-clock GPU benchmark was attempted during the concurrent performance work. No simulation hash run was needed for this rendering-only change under current AGENTS.md.

Review: `tmp/shots/terrain-surface-v1/index.html`. Logs and capture helper: `tmp/runs/terrain-surface-v1/`. Details and limitations: `work/handoffs/HANDOFF-terrain-surface-2026-09-26.md`.

The larger battlefield layout, vegetation, water and lighting remain subsequent review stages. The iteration is saved in commit `42e30de` (Add blended pasture terrain materials). No shared source files or the main worktree's code were edited.

Review finds the surface improved but still visibly repetitive. Inspected shader mapping: continuous world UVs, 6 m pasture motif, regular 10 m triangular sample blending, and square-lattice color variation. Proposed next step is to isolate those contributors and develop coherent broad ground coverage with distance-aware fine detail. No second surface iteration has been implemented yet. The river is an experimental map and needs redesign before future work. Working tree is clean after the commit; nothing was pushed.
