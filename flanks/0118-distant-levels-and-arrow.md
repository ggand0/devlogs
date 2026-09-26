# Authored distant levels and the arrow model

Written by Claude Opus 5.5.

2026-09-24. The last asset work on `feat/unit-models` before its PR.

## Distant levels

- Astra authored L2 (248 to 250 triangles), then L1 (664 to 680) and L3 (48 to 56), for all four kinds. Each is appended to the installed GLB with the same atlas, part ids and pivots. The loader uses every level a file carries, so the derived boxes are gone.
- Commits: 8d56111 knight L2, b1e1e03 the other L2s, bab8abb every L1 and L3, plus c1f6494 and 771412e for the builders in `tools/blender/lods/`.
- I had called authored L1/L3 unnecessary without looking. Rendered side by side (`tmp/shots/v3/lod_levels.png`), the derived L1 boxes looked nothing like the models. The authored ones do (`levels_arch.png`, `levels_maa.png`).
- Open: the archer's L3 renders grey rather than blue up close. It is only drawn under 3 px. Check that far archer regiments still read blue.

## Arrow

- `unit_glb::arrow_mesh` loads `assets/units/arrow.glb` (or `FL_GLB_ARROW`) and falls back to the code-built arrow.
  - The model is authored at true size, 0.76 m.
  - It is scaled to the code-built arrow's length, 0.79 in the metre world, where soldiers stand about 1 m tall. Volleys read as before.
- Commits: 7986911 loader, a63428e model, ab2aa0c Astra's build scripts.

## Checks

`dir` and `arch` fingerprints are 19/19 against the pipelined-18ada02 baselines after all of it, and clippy is clean.

## Next

- PR for the branch.
- The knight's diagonal slash (`work/notes/knight-diagonal-slash-v4-for-claude-2026-09-24.md`) goes to its own branch. Before it plays in game:
  - a shader port of `tools/blender/melee/sword_slash.py` `deform`
  - a three-way style pick for knights, which changes the fingerprints because swing bytes are hashed
  - blending into and out of the clip's own ready pose
  - Astra porting the cuff fix into the four-level knight
- Astra's untracked slash files (`tools/blender/melee/`, `tools/blender/knight/*slash*.py`) belong to that branch.
