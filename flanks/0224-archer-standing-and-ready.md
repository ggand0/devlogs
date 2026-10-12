# Archer standing and ready pose pair

Written by GPT-6.

Gota accepted the tube-leg revision and requested standing and ready poses. Both existing 22-joint clips are now presented explicitly at the top of lower_body_v3/review.html with direct JSON links. Standing uses the delivered upright rest pose, ready uses the grounded staggered stance. No pose rotations or model geometry changed in this turn.

Rendered standing from seven directions and inspected it beside ready. Standing width 0.787200 m; ready width 0.787148 m. Both hide arrow part 16 in the review. Each file has one 91-value sample with the same 22 joint names. Prior shared clip validation passes for each. Checked local page links, image decoding and clip structure; unchanged diagnostic playback retains the prior successful full-loop check.

Archive the accepted lower_body_v3 iteration and its README rows in the private assets_dev repository. Earlier reshaped-leg iterations remain on disk. No src or shipped asset changes. Lower-detail cloth remains the source data; this is an L0 model/pose review, not completion of all four levels or the archer shot integration. The pair is ready for review at assets_dev/archer/lower_body_v3/review.html.

Private archive commit: 81a393615543d3cc4418861baebd17483f1539c9. Seven media files are verified in Git LFS; GLB, atlas, regions and frame sequences stay on disk. No push.

## Restore both orbits and give the pose review its own folder

Gota pointed out that the pose review had lost the orbit and was still named lower_body. The earlier leg-rework review had only a fixed-view diagnostic sweep, which did not replace the ready_v1 orbit. Created archer/standing_ready_v1 with a standalone copy of the build script, the same model and pose outputs byte for byte, and new render/compose/check scripts. lower_body_v3 and earlier iterations remain in place.

Each pose has 576 new 640-square frames at 24 fps: a full orbit in one direction for 12 seconds, then back in the other direction for 12 seconds, easing at the reversal. Each pose stays fixed throughout. The new page opens with the two orbit videos, controls and direct clip links; the pair and seven-direction sheets follow. No model, pose rotations, gameplay source or shipped files changed.

Both orbit videos passed full decoding, 576-frame/24-second checks, all-angle framing and complete-loop playback in an isolated browser. Inspected the ordered orbit sequences and browser presentation. New private archive commit: ab5b099b087320cfef0ba0c3c5b636f129d8cbfa. All PNG/MP4 files use Git LFS.

## Restore the missing M2TW correspondence

Gota asked where the reference images and correspondence had gone. Ready_v1 had a peasant_archers:ready:0 comparison, but the later lower-body and pose-review pages omitted it. The earlier record confirms that ready and standing reference sheets were inspected, ready's GIF was inspected as ordered frames, and ready was measured. Standing was preserved from the model's rest pose; it did not receive its own matching stand_a_idle comparison in those iterations. The broad pose-validation statements referred to clip/geometry validation, not reference agreement.

Reran the shared number check on the current 22-joint model and saved stand/ready JSON files. Ready still has 16 of 17 rows within tolerance; bow axis is 46.2 degrees away because the brief keeps the stave upright. Standing against peasant_archers:stand_a_idle:0 has 3 of 17 rows over: bow axis 49.4 degrees, foot spacing 15.8 percentage points of leg length (ours 0.28, reference 0.44), and right/draw forearm 17.1 degrees. Standing's body difference is now explicit; no pose or geometry changed in response to this check.

Built local_reference/ with exact frame-0 pairings, M2TW above ours across all seven matching camera views, both at 2.2 m view width. Inspected both full reference sheets, the paired rendered rows and every reference frame from the right side (29 stand frames, 21 ready frames). Full reference GIFs and complete sheets are linked alongside current measurement tables. Added source/pose/model hashes in the ignored local manifest. The existing number-check diagram puts a weapon line at the right hand even for an archer, so this revision uses the rendered comparison rows and numeric bow-axis measurement, not that misleading diagram.

Main review embeds both comparison images and links to the detailed local page; both orbits remain. Browser image loading, links and source hashes pass. Checked that every local_reference output is ignored before staging only review scripts, HTML, README and .gitignore. M2TW imagery and measurements are not committed. The missing correspondence was a review-page omission, not evidence that the later standing pose had been matched to the reference.
