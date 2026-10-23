# Wall thickness reduced to 8 m

Written by GPT-6.

2026-10-23. Gota approved the first texture version and requested 8 m wall thickness. Built `assets_dev/wall/textured_v2/` with a 2 m length and 8 m thickness; preserved both earlier iterations. No new texture source generation. Physical texture scale is unchanged, with a fresh atlas bake for the narrower surface.

The normal wall has a 6 m clear walkway at height 8 m, between two 1 m edge strips. The landing exposes 7 m of top surface with the inner kerb removed. Both footprints are X [-1, 1], Z [-4, 4]. Total height remains 9.5 m, merlon centre X = 0, pitch 2 m. Wall 150 triangles, landing 134. Both GLBs pass exact end matching, closed-surface and winding checks, atlas embedding identity and per-triangle texture coverage. The standard GLB attribute inspection also passes. A fresh set of eight renders and the review sheet is in the new folder.

The default builder thickness is 8 m and the validator independently expects 8 m. Updated review captions and the close-view camera target. The original 10 m textured version was archived on assets_dev main with source and review images in LFS, as required after review. Existing unrelated work and README edits were preserved.

Plan 028 now records the requested dimensions and explicitly supersedes handoff 145's wall thickness and clear-walkway figures. The tower is still 16 m across with the previously specified 8 m passage; its connection to the narrower 6 m wall walk needs checking before building. X = 7 m remains a safe first-unit centre: at Z = +/-4 m, the radius-8 cylinder reaches X = sqrt(48), about 6.928 m, while the unit starts at X = 6 m. Stair width is unchanged; its placement must use the new inner face at Z = -4 m.

Recommended next step is a short in-game wall run with actual soldiers to judge the 6 m clear width and camera appearance. This needs matching model placement and the 8 m raised-terrain footprint. Engine integration is not implemented here and needs its own scope and shared-file coordination. After width review, build the drum tower, then the stair, with individual reviews, and finally check the assembled route and repetition in-game. Do not run those stages unattended.

No engine or sim edits, no game launch, no build or hash test. Checked for other games before the headless Blender run. Review the updated 8 m render before proceeding.
