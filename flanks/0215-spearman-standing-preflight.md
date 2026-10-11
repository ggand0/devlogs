# Spearman standing pose preflight

Written by GPT-6.

2026-10-11. Gota requested the standing pose first and a review before proceeding to ready. Read handoff 102 and its workflow, skeleton, asset, reference and worktree documents. Worked from the graphics tree root; no branch change.

The promised spearman skeleton folder is still absent. The shipped graphics-tree model and the latest four-level private model are identical, unskinned files. Measured widths are 0.998397 / 0.997082 / 0.996373 / 0.970792 m, with 2992 / 664 / 250 / 56 triangles. Standing L0 is 98.4 mm over the 0.9 m width requirement. The weapon hand and shield-arm part define the two lateral extrema. Ground minimum is zero at every level; L0 forward extent is 0.294288 m.

Inspected both seven-view standing reference sheets and the existing spearman level review. Measured static reference frame 0 only. Identified shared-tool assumptions that need resolving with the rig: the pose checker reconstructs the knight's rig/skin, armored-sergeant reference weapon extraction returns None, the weapon-aim helper assumes a +Z rest blade and is singular at exactly upright, and helper validation is restricted to the knight skeleton name.

Detailed measurements and requested prerequisites are in `work/notes/051-spearman-standing-prerequisites-2026-10-11.md`. Gota has been asked whether Claude will deliver the prerequisites or this task may take them over. No standing candidate or approval review is ready yet. No model, clip, shared tool or game source was modified; no build, Blender or game was run. No commits were made. GIF motion was not watched.
