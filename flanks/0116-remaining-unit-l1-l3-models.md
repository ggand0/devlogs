# Author L1 and L3 for the remaining units

Written by GPT-6 Astra.

2026-09-24. Gota approved the knight L1/L3 and authorized all remaining kinds using the same approach.

## Assets

| Unit | L0 | L1 | L2 | L3 |
|---|---:|---:|---:|---:|
| man_at_arms | 2998 | 674 | 248 | 48 |
| spearman | 2992 | 664 | 250 | 56 |
| archer | 2987 | 680 | 250 | 56 |

Candidates and review files are in `assets_dev/<kind>/lod_l1_l3_v1/`. The combined comparison page is `assets_dev/lod_l1_l3_review/index.html`.

L1 uses closed, shaped torso/limb masses and retains the animated part IDs, original pivots, equipment outlines, kettle hats and bow limbs. The archer keeps both complete arm chains, torso/head, quiver and arrow fan; the held arrow and strings are omitted. Its cape has a separate atlas category and whole-figure visibility correction after fitting the individual parts.

L3 keeps all vertices on part 0 with height 0. It retains the coarse head, body and leg masses plus shield, spear line or bow. Each level samples the original embedded atlas, including team amount.

`build_unit_l1_l3.py`, `infantry_levels.py`, `archer_levels.py` and the compact source label manifests reproduce the models. `source_base.py` recovers the original L0/L2 input from an installed four-level file using preserved JSON metadata, binary length and SHA-256. Rebuilding from each completed candidate produced byte-identical GLBs and sidecars.

## Verification

Both GLB inspectors, surface inspection and `verify_levels.py` pass for all candidates. All new surfaces are closed with zero nonmanifold, same-winding or degenerate geometry. VEC4 colours and TEXCOORD_1 part/pivot values survive the specified export. Original L0/L2 bytes, nodes, materials, embedded atlas and spearman/archer sidecars remain unchanged. Existing L0 topology defects are recorded separately in the validation reports.

Head height is 1.80 m, with the spearman tip at 2.52 m. Separate 0–60 degree pivot sweeps pass for every checked joint. Sampled attack sequences cover 90 frames for man-at-arms, 64 for spearman and 150 for archer, with no disconnected checked joints. Reviewed multi-frame contact sheets and static comparisons; videos are provided at the matching 15/20/15 fps.

Compared L0/L1/L2/L3 at 20, 8 and 3 px, from front/back/side, plus red and blue team views. The maximum mean RGB channel difference against L0 is 0.99%/0.93% for man-at-arms L1/L3, 1.72%/1.80% for spearman, and 0.36%/1.27% for archer. Team coverage is recorded in each comparison report.

## Handoff

`work/notes/remaining-unit-l1-l3-for-claude-2026-09-24.md` contains paths, counts, per-part heights, hashes, preservation checks and installation guidance. The previously approved knight note is `work/notes/knight-l1-l3-for-claude-2026-09-24.md`.

Stopped with reviewable candidates. No installed asset, src/ or renderer edits; no commits or staging. Other concurrent knight animation work was left untouched.
