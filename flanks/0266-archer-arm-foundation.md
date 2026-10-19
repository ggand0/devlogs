# Archer arm foundation
Written by GPT-6.

The shoulder-cloth revision was rejected because it removed required shoulder coverage while keeping the underlying ball-and-tube anatomy. The hood was also too large. The requested order is now explicit: rebuild the entire arms before covering the result with the cape, then review. The cape must ultimately cover the shoulders and the hood must fit closer to the head.

The original reference was inspected again. It shows a narrow shoulder line under loose upper sleeves, gathered linen at the elbows, tapering forearms and small wrists, with a fitted left bracer. The source geometry explicitly builds separate icosahedral joint covers and short tubes. The knight skeleton and its sleeve weighting were inspected as the house reference.

## Work

Created assets_dev/archer/arms_v1. Removed 813 source L0 triangles: upper tunic, all arm geometry, shoulder balls and shoulder cape. Rebuilt the upper tunic and both sleeves as one connected 640-triangle surface, preserving the exact source waist boundary. The body-to-arm openings are stitched directly into the sleeve topology. The anatomical sections change through the shoulder, biceps, elbow, forearm and wrist; the upper sleeve ends above the linen forearm. The left bracer follows that forearm shape. Each hand is a closed 92-triangle surface with a connected thumb web.

The existing 22-joint skeleton is unchanged. Shoulder-root, elbow and wrist blends use only parent/child pairs. Numeric checks exposed underarm loop folding, elbow compression, wrist torsion and thumb self-intersection during development. Geometry and weighting were corrected, including the forearm section frame and the distribution of wrist rotation. The final new surfaces have no self-intersections in the 174 sampled poses. No changes were made to the clips.

The source coif stays for context in this arm review. The cape is removed in both before and after previews. Hood v1 and v2 remain on disk and uncommitted. No new hood or cape is being presented as finished.

## Measurements

- L0/L1/L2/L3 triangles: 2,998 / 680 / 250 / 56.
- Height: 1.80 m. All joint rest positions, pivot nodes, materials and retained vertex attributes match the source.
- New surface triangles: 824. Components: 640, 92, 92. All watertight and positive-volume; zero open edges, nonmanifold edges, winding conflicts or degenerate triangles.
- 174 pose samples: stand 1, ready 1, draw 61, aim_up 61, release 25, release_up 25. Maximum shared-vertex seam error is zero; no new-surface self-intersections.
- Arms/head and sleeves/bow-limbs: zero sampled crossings.
- The original early-draw string crossing remains at 0.125, 0.150, 0.175 and 0.200 seconds. The source has 24–26 crossings at those times; the new mesh has 12–14. Both releases are clear. The clip itself remains unchanged.
- All distant meshes are byte-for-byte attribute matches to the source. The final LOD work waits for the near-shape review.

The candidate is not installed or committed. Review files are in the iteration folder: review.html, clay_comparison.png, comparison.png, poses.png, arms_motion.mp4 and arms_orbit.mp4. Scripts remain there. GPU work uses the host game guard before each Blender invocation.

Review verification completed: all local links resolve, all 816 frames decode, and both the 10-second fixed view and 24-second orbit play through a loop in headless Firefox. Playback captures were inspected across drawing, elevated aim, release and both camera directions. The orbit uses one fixed 2.143 m vertical scale fitted to the full moving-arm envelope; maximum normalized horizontal extent is 0.874.

Final candidate SHA-256: a2298fedf662d5606485ab1f77af5aaaba381c2ffe4af420085eccaf552f87f8.
