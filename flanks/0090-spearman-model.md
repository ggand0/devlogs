# 0090: textured spearman, model height fix, tooling review (2026-09-23)

Branch `feat/knight-model`.

- `6a8a3a9` Astra's knight build scripts in `tools/blender/`, amended in place (backup `backup/knight-model-f3a5913`, bundle tmp/backups/knight-model-2026-09-23c.bundle). The README credited "the owner" twice, a docstring said "never the user's live scene", and several comments described earlier review rounds ("like the v1 bucket", "no oversized bright lower rim", "not fantasy horns"). Those now state what the code does.
- `884248e` The loader measured a model's height as its highest vertex. The spearman's upright spear reaches 2.52 m, so he came out at 70 per cent size. Height is now the top of the body part, with the whole mesh as the fallback. The knight's scale is unchanged (0.611).
- `f07ab59` Astra's spearman v2 (face rework: nose, philtrum, eyes, lips, stubble) promoted to `assets/units/spearman.glb` through LFS. 2,998 tris, one 2048 px atlas, spear arm part 4 with `pivot_arm_spear`, shield arm part 5. Checked in game marching and closing: scale right, face, coif and quilting read, spears come down as the block advances.

Open: the spear arm pitches as one rigid part about the shoulder, spear and arm together. With a realistic arm the levelled pose swings the whole arm, not just the spear. A proper hold needs an elbow or a wrist pivot from the asset side, which belongs with the upper body work in devlog 0088.
