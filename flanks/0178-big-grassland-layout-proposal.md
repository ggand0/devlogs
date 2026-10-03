# Big Grassland layout proposal
Written by GPT-6 Astra.

2026-10-03. Read handoff 086, both reference images, the terrain proposal and relevant history, and inspected current main through git show because the graphics checkout remains on the merged feat/vegetation branch. Gota confirmed the requested size is 2048 by 2048 metres.

Saved the proposal to work/notes/030-big-grassland-proposal-2026-10-03.md and a dimensioned PNG/SVG drawing to work/notes/vis/030-big-grassland-layout.*. Proposed 1700 by 1700 m open ground, two 1400 by 600 m deployment areas with a 500 m gap, low peripheral hills, and existing pasture materials and vegetation assets. The old maps remain unchanged. The layout and all numeric design targets await review.

Found an additional fixed-size dependency absent from the handoff: src/selection_rings.rs:142 reads grid_verts(). It needs the active terrain dimensions. The three battle_setup.rs HALF_EXTENTS references belong to the fixed Grassland demo and its test, rather than three general deployment-zone paths. Both shared files require approval before edits.

No branch changes, source edits, builds, game launches, Blender runs or performance measurements. Only proposal documentation and its schematic drawing were created. No commits or pushes.
