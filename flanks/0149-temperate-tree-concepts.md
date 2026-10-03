# 0149: Temperate tree concepts
Written by GPT-6 Astra.

2026-09-27. Branch feat/vegetation, graphics worktree.

Gota accepted oak v2 for game use. He approved the remaining branch scope: two more oak variants, one contrasting birch-like tree, two shrub forms, then placement, wind and the populated performance check. Dense meadow ground cover belongs to a separate branch, with asset art here and Claude handling its population renderer and cost checks. The next step is concept review before constructing the three new trees.

Generated one reference sheet per new tree using the built-in image_gen tool, with the accepted oak render and generated grassland reference as inputs. All three images are saved unchanged under work/notes/022-vegetation-tree-concepts-2026-09-27/. The directory includes index.html, exact prompts in prompts.json and image checksums in sources.json.

- Mature oak: proposed 14 m height and 13 m crown width, substantial crooked branches and an uneven broad crown.
- Smaller leaning oak: proposed 9 m height and 6.5 m width, gentle trunk lean, lighter branch structure and asymmetric crown.
- Birch companion: proposed 16 m height and 6 m width, pale marked trunk, smaller leaves and more open foliage with fine drooping shoots.

These dimensions are authoring targets, not measurements from an asset. The reference sheets are independently framed and not at a shared scale. Their second views suggest crown volume but are not geometrically verified turnarounds. The oak images contain warmer highlights than the accepted atlas; use the accepted oak's neutral green palette when building their meshes. No new tree models, textures for shipping, integration edits or performance measurements were made in this step.

The three silhouettes are distinct enough to review before building: broad mature mass, smaller lean, narrow pale-trunk companion. One generated candidate per tree was retained without a tuning loop. The existing public vegetation diff remains unchanged. No builds, Blender runs or games were launched. Stop at concept review before modeling.

## Birch concept revision

Gota approved mature oak A and smaller leaning oak B. B remains a 9 m tree, compared with the accepted upright oak at 13.164 m; the sheets are independently framed for detail. Birch C was intended as silver birch, Betula pendula. Gota supplied wikipedia_birch.jpg for discussion, then preferred the upright crown of reddit_silver_birch.webp and requested a slightly slimmer interpretation.

Converted the Reddit WebP to resources/vegetation/reddit_silver_birch.jpg at its original 1080 × 1080 resolution, preserving the original. This photo remains a private shape reference. The revised birch concept, birch-v2.png, follows its exposed slender trunk, connected uneven crown and upward/outward branches. Long hanging tiers are removed; small outer shoots retain some natural droop. Proposed model proportions are 15–16 m tall and about 8 m wide, rather than the earlier 6 m width. These remain targets, not measurements. The image generator was asked for a modest 10–15% narrowing relative to the photo, not a columnar tree.

One built-in image_gen edit produced the retained concept. Exact prompt and references are in birch-v2-prompt.json. The comparison page shows revised C, preserves a link to original C, and marks A/B approved. No models or source code changed. Revised C remains for visual review before modeling.

## Shrub reference concepts (2026-09-27)

After accepting warm birch, Gota requested bush design/reference images while Claude implements an empty map on this branch. This is the previously agreed two-shrub stage, with dense meadow cover still separate. No game, Blender, compile or runtime edits are needed for concept work.

Inspected resized debug0.png (M2TW scattered bushes), debug1.png (dense meadow vegetation, outside this stage), and grassland_map_ref.png. Proposed two hawthorn-like scrub forms sharing a leaf/material family: upright 1.7 m high by 2.2 m wide, and low spreading 0.8 m high by 2.5 m wide. Upright shrub provides accents and loose broken hedges; spreading scrub fills smaller isolated patches and the bases of larger plants. These are art directions, not botanical identifications of the screenshots.

Used the built-in image_gen tool via the imagegen skill, one text-only generation per form. Two original reference sheets are saved in assets_dev/vegetation/shrub_concepts_v1/, with exact prompts, original generated-image paths, hashes and dimensions. No reference photograph or game screenshot pixels were used. Review page: work/notes/024-vegetation-shrub-concepts-2026-09-27/index.html.

Both sheets have believable multi-stem wood, irregular connected leaf masses, small gaps and restrained warm greens. The upright sheet is dense with sparse upper shoots; the spreading sheet has outward-growing low branches and tapering sprays. Preserve these forms when modeling, but simplify the visible fine twig network into leaf sprays and retain only principal woody branches as geometry. Proposed caps: upright 600/160/4 triangles, spreading 420/120/4. Shared 1024-square foliage colour texture, separate opaque bark texture, far colour/normal bakes up to 512-square per form. These are targets, not verified mesh counts or performance results.

Sheets are independently framed, so B's target height is less than half A's despite both images filling their page. Elevated views are illustrative, not verified geometry. Final material colour will need in-game review alongside the accepted trees from the sun-facing side. One candidate retained per form, no tuning loop. Stop at reference approval before models; no empty-map implementation or code edits made in this stage.

Private concept iteration commit: 6c36ab1. Both original concept PNGs were verified as Git LFS pointers before committing. Review links checked. Public code changes seen in src/game_state.rs and src/regiments.rs belong to concurrent work and were left untouched.

## B selected for modeling

Gota accepted the concepts and selected low spreading B first, then authorized building it and reviewing it in the empty Scene scenario beside an oak and warm birch. The first model, budgets, validation and review are recorded in devlog 0154. Organic grouping remains the following stage.
