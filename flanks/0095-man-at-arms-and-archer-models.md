# Man-at-arms and archer models in game

Written by Claude Opus 5.5.

2026-09-23. Astra's textured man-at-arms v2 and archer v2 now replace the code-built meshes.

## Man-at-arms v2

Committed as 3946b7a. An arming sword and a heater shield on the spearman body. The model is 2,980 triangles and 1.80 m tall, with a 2048² atlas, and it loads without warnings.

The sword is part of `arm_weapon`. The model has no weapon part and no elbow empty, so the loader gives it no rig: the arm and sword turn as one piece about the shoulder, as the code-built man-at-arms did before the bent arm. It cannot detach. A jointed stab like the spearman's needs a forearm, hand and weapon split plus sword tables for both swing styles.

## Archer v2

Installed at `assets/units/archer.glb`, not yet committed. The face was rebuilt. The model is 2,985 triangles with a 2048² atlas and loads without warnings. Parts: body, `arm_weapon` (draw arm), legs and `arm_bow`. It has no weapon or jointed parts, so the draw arm runs the old rigid pull-back-and-release, and the bow arm tilts toward the loft angle as before. Checked in `FL_TEST_ARCHERY`: faces, coifs, quivers and bows read at close range, and the bows lift on the draw.

## Model sizes

Each model file is 5.0 to 5.9 MB. About 97% of that is the embedded 2048² PNG atlas: 4.8 to 5.7 MB per kind. The geometry is about 0.2 MB.

## Shield back (open)

All three shielded kinds share a heater shield that is a one-sided shell facing forward. Below the arm it has 0.071 m² facing forward and 0.001 m² facing back, so it is see-through from behind under back-face culling. The fix is in the model: a closed back board with its own texture. The note for Astra is `work/notes/astra-shield-back-2026-09-23.md`.

## Capture script

`work/scripts/clip.sh` found the window by its title only, and it recorded a file manager showing a folder named flanks. It now requires the game's pid as well as the title.
